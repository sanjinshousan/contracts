# system合约
					

## exchange_state.cpp

以eos为基础币，发行的2种类型的代币。一个账户的usd代币，给自己的btc代币转换100个usd调用这个cpp	
例如一个dan 0 eos。   supply 1e+11 eos
		dan 100 usd     quote	1e+8 usd
		dan 0 btc		 base 1e+8 btc


```
 // 给出当前的state，计算一个新的state回来
   asset exchange_state::convert( asset from, symbol_type to ) {
      auto sell_symbol  = from.symbol;	  // usd
      auto ex_symbol    = supply.symbol; // eos
      auto base_symbol  = base.balance.symbol; // btc
      auto quote_symbol = quote.balance.symbol;  // ust

      //print( "From: ", from, " TO ", asset( 0,to), "\n" );
      //print( "base: ", base_symbol, "\n" );
      //print( "quote: ", quote_symbol, "\n" );
      //print( "ex: ", supply.symbol, "\n" );

      if( sell_symbol != ex_symbol ) {     // 币的名字不相同
         if( sell_symbol == base_symbol ) {     // 如果卖出者的币名与base的币名相同
            from = convert_to_exchange( base, from );
         } else if( sell_symbol == quote_symbol ) {
            from = convert_to_exchange( quote, from );
         } else { 
            eosio_assert( false, "invalid sell" );
         }
      } else {
         if( to == base_symbol ) {               
            from = convert_from_exchange( base, from ); 
         } else if( to == quote_symbol ) {
            from = convert_from_exchange( quote, from ); 
         } else {
            eosio_assert( false, "invalid conversion" );
         }
      }

      if( to != from.symbol )
         return convert( from, to );

      return from;
   }
   
```

sell_symbol 与 基础币不相同，调用conver_to_exchange 转化为eos代币。		
然后to 与from 不相同，在继续调用conver_from_exchange 转为btc代币


```
asset exchange_state::convert_to_exchange( connector& c, asset in ) {

	  real_type R(supply.amount);  // 1e + 11;
     
      real_type C(c.balance.amount+in.amount);  // state资产新发行的代币。
     
      real_type F(c.weight/1000.0); // 代币所占比重,初始化设置好的
      
      real_type T(in.amount);// 新发行数量 100
      real_type ONE(1.0);
    	// 根据这个算法得到对应的state资产的增发量。
	   real_type E = -R * (ONE - std::pow( ONE + T / C, F) );
	   
      //print( "E: ", E, "\n");
      
      int64_t issued = int64_t(E); // 大概是48999 ，增发100个usd，实际上要增发state这么多
      
      
      supply.amount += issued;	// 更新总发行量，然后某稳定数字资产下的token余额增加了
      c.balance.amount += in.amount;  //
       
       return asset( issued, supply.symbol );  // 以eos资产实际上增加的形式返回.
```


	

```
asset exchange_state::convert_from_exchange( connector& c, asset in ) {
      eosio_assert( in.symbol== supply.symbol, "unexpected asset symbol input" );

      real_type R(supply.amount - in.amount);  // 先找回原值，1e+11;
      real_type C(c.balance.amount);            // btc 余额不变，仍为1e+8
      real_type F(1000.0/c.weight);
      real_type E(in.amount);			// 489999
      real_type ONE(1.0);           


     // potentially more accurate: 
     // The functions std::expm1 and std::log1p are useful for financial calculations, for example, 
     // when calculating small daily interest rates: (1+x)n
     // -1 can be expressed as std::expm1(n * std::log1p(x)). 
     // real_type T = C * std::expm1( F * std::log1p(E/R) );
      
      real_type T = C * (std::pow( ONE + E/R, F) - ONE); //大概是96
      //print( "T: ", T, "\n");
      int64_t out = int64_t(T);

      supply.amount -= in.amount; // 将eos增发的部分减掉，维持原来的1e+11
      c.balance.amount -= out;	// btc的总量减少了96.

      return asset( out, c.balance.symbol );
   }
   
```

最终的结果：
	dan 0 eos supply 1e+11
	dan 0 usd base 1e+8usd
	dan 96 btc quote 9.999e+7btc




