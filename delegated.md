# delegate_bandwidth


##用户资源
```
struct user_resources {  // 
      account_name  owner;
      asset         net_weight;
      asset         cpu_weight;
      int64_t       ram_bytes = 0;
      asset         governance_stake;
      time          goc_stake_freeze = 0;

      uint64_t primary_key()const { return owner; }

      // explicit serialization macro is not necessary, used here only to improve compilation time
      EOSLIB_SERIALIZE( user_resources, (owner)(net_weight)(cpu_weight)(ram_bytes)(governance_stake)(goc_stake_freeze) )
   };
```

##用户抵押记录

```
struct delegated_bandwidth { 
      account_name  from;
      account_name  to;
      asset         net_weight;
      asset         cpu_weight;

      uint64_t  primary_key()const { return to; }

      // explicit serialization macro is not necessary, used here only to improve compilation time
      EOSLIB_SERIALIZE( delegated_bandwidth, (from)(to)(net_weight)(cpu_weight) )

   };
  
```

##用户赎回记录
```

struct refund_request {    //记录赎回记录
      account_name  owner;
      time          request_time;
      eosio::asset  net_amount;
      eosio::asset  cpu_amount;

      uint64_t  primary_key()const { return owner; }

      // explicit serialization macro is not necessary, used here only to improve compilation time
      EOSLIB_SERIALIZE( refund_request, (owner)(request_time)(net_amount)(cpu_amount) )
   };
   
```

## 函数

	其中比较核心的几个地方是 #define CORE_SYMBOL S(4,SYS) 是
							 S(4,RAMCORE)是中间代币
							 S(0,RAM) 是ram
* system_contract::buyram
	
```
//根据当前市场的份额，将需要购买的字节数转化为指定的EOS进行购买
void system_contract::buyrambytes( account_name payer, account_name receiver, uint32_t bytes ) {
      //在数据库中查询RAMCORE发行量，默认为100000000000000
      auto itr = _rammarket.find(S(4,RAMCORE));
      auto tmp = *itr;
      auto eosout = tmp.convert( asset(bytes,S(0,RAM)), CORE_SYMBOL );
      //通过转化后，调用buyram使用EOS购买
      buyram( payer, receiver, eosout );
   }
```
RAM的交易机制采用Bancor算法，使每字节的价格保持不变，通过中间代币(RAMCORE)来保证EOS和RAM之间的交易流通性。从上源码看首先获得RAMCORE的发行量，再通过tmp.convert方法RAM->RAMCORE，RAMCORE->EOS(CORE_SYMBOL)再调用 buyram 进行购买。这里的CORE_SYMBOL不一定是指EOS，查看core_symbol.hpp，发现源码内定义为SYS，也就是说在没有修改的前提下，需要提前发行SYS代币，才能进行RAM购买。

* buy

```
void system_contract::buyram(account_name payer, account_name receiver, asset quant)
{
      //验证权限
      require_auth(payer);
      //不能为0
      eosio_assert(quant.amount > 0, "must purchase a positive amount");

      auto fee = quant;
      fee.amount = ( fee.amount + 199 ) / 200;
      //手续费为0.5%，如果amoun为1则手续费为1，如果小于1，则手续费在amoun<free<1 quant.amount.
      //扣除手续费后的金额，也就是实际用于购买RAM的金额。如果扣除手续费后，金额为0，则会引起下面的操作失败。
      auto quant_after_fee = quant;
      quant_after_fee.amount -= fee.amount; the next inline transfer will fail causing the buyram action to fail.

      INLINE_ACTION_SENDER(eosio::token, transfer)
      (N(eosio.token), {payer, N(active)},
       {payer, N(eosio.ram), quant_after_fee, std::string("buy ram")});

      //如果手续费大于0，则将手续费使用eosio.token合约中的transfer，将金额转移给eosio.ramfee，备注为ram fee
      if (fee.amount > 0)
      {
            INLINE_ACTION_SENDER(eosio::token, transfer)
            (N(eosio.token), {payer, N(active)},
             {payer, N(eosio.ramfee), fee, std::string("ram fee")});
      }

      int64_t bytes_out;

      //根据ram市场里的EOS和RAM实时汇率计算出能够购买的RAM总量
      const auto &market = _rammarket.get(S(4, RAMCORE), "ram market does not exist");
      _rammarket.modify(market, 0, [&](auto &es) {
            //转化方法请参考下半部分
            bytes_out = es.convert(quant_after_fee, S(0, RAM)).amount;
      });

      //剩余总量大于0判断
      eosio_assert(bytes_out > 0, "must reserve a positive amount");

      //更新全局变量，总共可以使用的内存大小
      _gstate.total_ram_bytes_reserved += uint64_t(bytes_out);
      //更新全局变量，购买RAM冻结总金额
      _gstate.total_ram_stake += quant_after_fee.amount;

      user_resources_table userres(_self, receiver);
      auto res_itr = userres.find(receiver);
      if (res_itr == userres.end())
      {
            //在userres表中添加ram相关记录
            res_itr = userres.emplace(receiver, [&](auto &res) {
                  res.owner = receiver;
                  res.ram_bytes = bytes_out;
            });
      }
      else
      {
            //在userres表中修改ram相关记录
            userres.modify(res_itr, receiver, [&](auto &res) {
                  res.ram_bytes += bytes_out;
            });
      }
      //更新账号的RAM拥有量
      set_resource_limits(res_itr->owner, res_itr->ram_bytes, res_itr->net_weight.amount, res_itr->cpu_weight.amount);
}

```

* sellram 

1.同样的，调用convert方法，将所出售的ram的字节数，根据市场价格换算为EOS的数量(tokens_out变量来表示),修改表格。

2.更新全局变量

* delegatebw

```
 //抵押token，换取网络与cpu资源
 //from是抵押者，receiver是token的接收者，大多数情况这俩个是同一个名字，但也可以抵押了把币给别人，
 //transfer是true接受者可以取消。在验证完数据的合法性，以后调用changebw方法
 
 void system_contract::delegatebw( account_name from, account_name receiver,
                                     asset stake_net_quantity,
                                     asset stake_cpu_quantity, bool transfer )
   {
      eosio_assert( stake_cpu_quantity >= asset(0), "must stake a positive amount" );
      eosio_assert( stake_net_quantity >= asset(0), "must stake a positive amount" );
      eosio_assert( stake_net_quantity + stake_cpu_quantity > asset(0), "must stake a positive amount" );
      eosio_assert( !transfer || from != receiver, "cannot use transfer flag if delegating to self" );

      changebw( from, receiver, stake_net_quantity, stake_cpu_quantity, transfer);
   } // delegatebw


```

*undelegatebw

```
// 取消抵押，释放网络和cpu。 from取消抵押取消抵押，receiver会失去投票权

void system_contract::undelegatebw( account_name from, account_name receiver,
                                       asset unstake_net_quantity, asset unstake_cpu_quantity )
   {
      eosio_assert( asset() <= unstake_cpu_quantity, "must unstake a positive amount" );
      eosio_assert( asset() <= unstake_net_quantity, "must unstake a positive amount" );
      eosio_assert( asset() < unstake_cpu_quantity + unstake_net_quantity, "must unstake a positive amount" );
      eosio_assert( _gstate.total_activated_stake >= min_activated_stake,
                    "cannot undelegate bandwidth until the chain is activated (at least 15% of all tokens participate in voting)" );

      changebw( from, receiver, -unstake_net_quantity, -unstake_cpu_quantity, false);
   } // undelegatebw
   
   
```   
* changebw

```
	1. 要求from的授权
	2.更新抵押记录表（更新del_bandwidth_table）
	3.更新抵押总量表（更新user_resources_table）
	4.更新退款表（更新refunds_table）
	5.若有需要，发送延迟退款交易
	6.更新选票权重

```


* * ***refund函数***：在***undelegateb函数***调用解除代币抵押后，将抵押的代币退回账户，会有个缓冲时间。

```c
void system_contract::refund( const account_name owner );
```

