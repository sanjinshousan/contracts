#system.cpp与system.hpp

## system.hpp

* eosio.system.hpp中定义了合约类eosiosystem::system_contract，和一些结构体，以及函数的申明。。		1. eosio_global_state 		全局状态
		2. producer_info				生产者信息
		3. voter_info 				投票人信息
		4. goc_proposal_info		goc提案信息
		5. goc_vote_info 			goc投票信息
		6.	goc_reward_info 			goc奖励信息
		
		
   
   
   * eosio_global_state , 定义了全局状态

```c
struct eosio_global_state : eosio::blockchain_parameters {
      uint64_t free_ram()const { return max_ram_size - total_ram_bytes_reserved; }

      uint64_t             max_ram_size = 64ll*1024 * 1024 * 1024;
      uint64_t             total_ram_bytes_reserved = 0;
      int64_t              total_ram_stake = 0;

      block_timestamp      last_producer_schedule_update;
      uint64_t             last_pervote_bucket_fill = 0;
      int64_t              pervote_bucket = 0;
      int64_t              perblock_bucket = 0;
      uint32_t             total_unpaid_blocks = 0; /// all blocks which have been produced but not paid
      int64_t              total_activated_stake = 0;
      uint64_t             thresh_activated_stake_time = 0;
      uint16_t             last_producer_schedule_size = 0;
      double               total_producer_vote_weight = 0; /// the sum of all producer votes
      block_timestamp      last_name_close;

      //GOC parameters
      int64_t              goc_proposal_fee_limit=  10000000;
      int64_t              goc_stake_limit = 1000000000;
      int64_t              goc_action_fee = 10000;
      int64_t              goc_max_proposal_reward = 1000000;
      uint32_t             goc_governance_vote_period = 24 * 3600 * 7;  // 7 days
      uint32_t             goc_bp_vote_period = 24 * 3600 * 7;  // 7 days
      uint32_t             goc_vote_start_time = 24 * 3600 * 3;  // vote start after 3 Days
      
      int64_t              goc_voter_bucket = 0;
      int64_t              goc_gn_bucket = 0;
      uint32_t             last_gn_bucket_empty = 0;
    };
```
* producer_info 定义了生产者的信息

```
struct producer_info {
      // 生产者节点的拥有者
      account_name          owner;
      // 获得的投票数
      double                total_votes = 0;
      /// a packed public key object 
      eosio::public_key     producer_key; 
      bool                  is_active = true;
      // 生产者的url
      std::string           url;
      // 生产的区块数量
      uint32_t              unpaid_blocks = 0;
      // 生产产出区块的时间
      uint64_t              last_claim_time = 0;
      // 生产者位置
      uint16_t              location = 0;

      // 主键通过owner索引
      uint64_t primary_key()const { return owner;                                   }
      // 二级索引通过投票数
      double   by_votes()const    { return is_active ? -total_votes : total_votes;  }
      bool     active()const      { return is_active;                               }
      void     deactivate()       { producer_key = public_key(); is_active = false; }

      // explicit serialization macro is not necessary, used here only to improve compilation time
      EOSLIB_SERIALIZE( producer_info, (owner)(total_votes)(producer_key)(is_active)(url)
                        (unpaid_blocks)(last_claim_time)(location) )
   };```
   

* voter_info 定义了投票人信息

```
struct voter_info {
      account_name                owner = 0; /// the voter
      account_name                proxy = 0; /// the proxy set by the voter, if any
      std::vector<account_name>   producers; /// the producers approved by this voter if no proxy set
      int64_t                     staked = 0;

      /**
       *  Every time a vote is cast we must first "undo" the last vote weight, before casting the
       *  new vote weight.  Vote weight is calculated as:
       *
       *  stated.amount * 2 ^ ( weeks_since_launch/weeks_per_year)
       */
      double                      last_vote_weight = 0; /// the vote weight cast the last time the vote was updated

      /**
       * Total vote weight delegated to this voter.
       */
      double                      proxied_vote_weight= 0; /// the total vote weight delegated to this voter as a proxy
      bool                        is_proxy = 0; /// whether the voter is a proxy for others


      uint32_t                    reserved1 = 0;
      time                        reserved2 = 0;
      eosio::asset                reserved3;

      uint64_t primary_key()const { return owner; }
   };
```
* goc_proposal _ info 定义了抵押过GOC token的用户提交propsal的信息

```c
struct goc_proposal_info {
      uint64_t              id;
      account_name          owner;
      asset                 fee;
      std::string           proposal_name;
      std::string           proposal_content;
      std::string           url;

      time                  create_time;
      time                  vote_starttime;
      time                  bp_vote_starttime;
      time                  bp_vote_endtime;

      time                  settle_time = 0;
      asset                 reward;

      double                total_yeas;
      double                total_nays;
      uint64_t              total_voter = 0;
      
      double                bp_nays;
      uint16_t              total_bp = 0;

      uint64_t  primary_key()const     { return id; }
      uint64_t  by_endtime()const      { return bp_vote_endtime; }
      bool      vote_pass()const       { return total_yeas > total_nays;  }
      //need change to bp count
      bool      bp_pass()const         { return bp_nays < -7.0;  }
   };
```

* goc_vote _info 定义了proposal投票的信息

```c
struct goc_vote_info {
     account_name           owner;
     bool                   vote;
     time                   vote_time;
     time                   vote_update_time;
     time                   settle_time = 0;

     uint64_t primary_key()const { return owner; }
   };

```
* goc_reward _info 对于BP出块的奖励信息

```c
struct goc_reward_info {
      time          reward_time;
      uint64_t      proposal_id;
      eosio::asset  rewards = asset(0);

      uint64_t  primary_key()const { return proposal_id; }
     };
```


##system.cpp



* void system_contract::setram( uint64_t max_ram_size )


	 	设置最大内存大小

* void system_contract::setparams( const eosio::blockchain_parameters& params ) 
	
		设置区块链参数
	

* void system_contract::setpriv( account_name account, uint8_t ispriv )

		设置账户权限
	
* void system_contract::rmvproducer( account_name producer )

		移除生产节点
	
* void system_contract::bidname( account_name bidder, account_name newname, asset bid )
	
		账号竞拍，对指定名字的账号进行竞拍，但有特殊要求，长度为0-11字符或者12字符且包含点
		新的竞价要比当前最高的竞价多出百分之10
   
* void native::newaccount( account_name     creator,
                            account_name     newact
                            /*  no need to parse authorities
                            const authority& owner,
                            const authority& active*/ )
                            
  		在创建新帐户后调用此Action，执行新账户的资源限制规则和命名规则



