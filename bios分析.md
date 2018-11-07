# bios分析

 **bios.cpp**主要功能在于控制其他账户的资源分配和权限
 
 	Head
 	-------------------------------------------------------
 	
 	
 
  * bios(action_name self):contract(self){}
  		
  		初始化合约名字
  		
  * setpriv(account_name account,uint8_t ispriv)

  		检测自己的权限，然后设置账号的权限。uint8_t 是unsigned char类型的，ispriv代表着级别
  		
  * setalimits
  		限制指定账户的所用的资源，内存大小，网络权重，cpu权重
  		
  * setglimits
  	
  		设置区块链的资源，无任何操作，暂时无效
  		
  * setprods(std::vector<'eosio::producer_key>schedule)
		
		设置区块链生产节点，
		schedule 参数是必要的检查
		
		read_action_data(buffer,size) 读取调用action所调用的size
		
		set_proposed_producers(bufffer,size) 设置生产区块的节点
		
		
		
  * reqauth(action_from)
  		
  		检测权限