
## <h2 id="1">背景</h2>   

**注意：V3.5版本后支持**

目前TRON的所有交易签名，都是用的同一个私钥完成。没有权限分级，也不能实现多人共同控制账户。因此，设计并实现多重签名功能，每个权限可以对应多个私钥。

[Tron multi-signatures Tip](https://github.com/tronprotocol/TIPs/issues/16)

## <h2 id="2">概念说明</h2>   

该方案共包含三种权限级别，owner、witness以及active权限，其中owner权限具有执行所有合约的权限，witness权限用于超级代表出块，active是自定义权限(可以组合权限集合)，以下将详细说明。

<h3 id="2.1">1）结构说明</h3>

**Account修改**
```text  
message Account { 
  ... 
  Permission owner_permission = 31;
  Permission witness_permission = 32;
  repeated Permission active_permission = 33;
}
```    
在账户结构中新增三个权限属性，分别是 owner_permission、witness_permission 和 active_permission，其中 active_permission 是个列表，可以指定最多8个。

**ContractType修改**
```text
message Transaction {
  message Contract {
    enum ContractType { 
      AccountCreateContract = 0; 
      ... 
      AccountPermissionUpdateContract = 46; 
    }
  }  
}
```     
新增一种交易类型 AccountPermissionUpdateContract，用于更新账户权限。

**AccountPermissionUpdateContract**    
```text
message AccountPermissionUpdateContract {
  bytes owner_address = 1;
  Permission owner = 2;   
  Permission witness = 3; 
  repeated Permission actives = 4; 
}
```     
`owner_address`：待修改权限的账户的地址  
`owner`：修改后的 owner 权限  
`witness`：修改后的 witness 权限（如果是 witness ）  
`actives`：修改后的 actives 权限  

该接口是覆盖原账户权限，因此，如果只想修改owner权限，witness（如果是witnss账户）及actives的也需要设置。
 
**Permission**
```text
message Permission {
  enum PermissionType {
    Owner = 0;
    Witness = 1;
    Active = 2;
  }
  PermissionType type = 1; 
  int32 id = 2;     
  string permission_name = 3;
  int64 threshold = 4;
  int32 parent_id = 5; 
  bytes operations = 6;  
  repeated Key keys = 7;
}
```
`PermissionType`: 权限类型，目前仅支持三种权限  
`id`: 值由系统自动设置，Owner id=0, Witness id=1, Active id 从2开始递增分配。在执行合约时，
通过该id来指定使用哪个权限，如使用owner权限，即将id设置为0。  
`permission_name`: 权限名称，由用户设定，长度限制为32字节  
`threshold`: 阈值，只有当参与签名的权重之和超过域值才允许做相应的操作。要求小于Long类型的最大值  
`parent_id`：目前只能为0  
`operations`：共32字节（256位），每位代表一个合约的权限，为1时表示拥有该合约的权限。 
如`operations=0x0100...00(十六进制),即100...0(二进制)`时,查看proto中Transaction.ContractType定义，合约AccountCreateContract的id为0，
即表示该permission只拥有执行AccountCreateContract的权限，可以使用"active权限中operations的计算示例"计算获得。  
`keys`：共同拥有该权限的地址及权重，最多允许5个key。  

**Key**     
```text
message Key {
  bytes address = 1;
  int64 weight = 2;
}
```
`address`：拥有该权限的地址     
`weight`：该地址对该权限拥有权重   

**Transaction修改**
```text
message Transaction {
  ...
  int32 Permission_id = 5;
}
```    
在交易中增加 `Permission_id`字段，与`Permission.id`相对应，用于指定使用哪个权限。默认为0，即owner权限。 
不允许为1，因为witness权限仅用于出块，不用于对交易进行签名。

<h3 id="2.2">2）Owner权限</h3>

OwnerPermission是账户的最高权限，用于控制用户的所有权、调整权限结构，Owner权限也可以执行所有合约。  

Owner权限具有以下特性：  
1、拥有OwnerPermission的地址可以修改OwnerPermission。  
2、当OwnerPermission为空时，默认采用该账户的地址具有owner权限。  
3、账户新建时，自动将该账户的地址填充到OwnerPermission中，并默认域值为1，keys中仅包含该账户地址且权重为1。  
4、当执行合约时未指定permissionId时， 默认采用OwnerPermission。  


<h3 id="2.3">3）Witness权限</h3>
超级代表可使用该权限，管理出块节点。非witness账户无该权限。  

使用场景示例：一个超级代表在云服务器上部署出块程序，为了账户安全，此时可以将出块权限赋予另一个地址。由于该地址仅具有出块权限，无TRX转出权限，即使该服务器上私钥被泄密，也不会出现TRX丢失。

Witness出块节点的配置：  
1、未修改witness权限时，无需特殊配置。  
2、修改witness权限后的出块节点，需要在重新配置，配置项如下：  

```text
#config.conf

// Optional.The default is empty.
// It is used when the witness account has set the witnessPermission.
// When it is not empty, the localWitnessAccountAddress represents the address of the witness account,
// and the localwitness is configured with the private key of the witnessPermissionAddress in the witness account.
// When it is empty,the localwitness is configured with the private key of the witness account.
//可选项，默认为空。
//用于当witness账户设置了witnessPermission。
//当该值不为空时，localWitnessAccountAddress代表witness账户的地址，localwitness是witnessPermission中的地址的私钥。
//当该值为空时，localwitness配置为witness账户的私钥。

//localWitnessAccountAddress =

localwitness = [
  f4df789d3210ac881cb900464dd30409453044d2777060a0c391cbdf4c6a4f57
]

```

<h3 id="2.4">4）Active权限</h3>

Active权限，用于提供一个权限的组合，比如提供一个只能执行创建账户、转账功能的权限。   

Active权限有以下特性：    
1、拥有OwnerPermission的地址可以修改Active权限    
2、拥有执行AccountPermissionUpdateContract权限的地址也能够修改Active权限    
3、最多支持8个组合。    
4、permission的id从2开始自动递增。    
5、账户新建时，自动创建一个Active权限，并将该账户的地址填充到其中，默认域值为1，keys中仅包含该账户地址且权重为1。  
 
 
<h3 id="2.5">5）费用</h3>

1、使用更新账户权限时，即 AccountPermissionUpdate 合约，收取100TRX。    
2、使用多重签名的交易时，即交易中包括两个及两个以上签名的交易，除交易费用外，另收取1TRX。    
3、可通过提议，修改以上费用。  

## <h2 id="3">API</h2>  

<h3 id="3.1">1）修改权限</h3>

`AccountPermissionUpdateContract`，修改权限步骤如下：  

1、使用接口`getaccount`查询账户，并获取原权限    
2、修改permission    
3、创建合约，签名    
4、发送交易    
 
**http-demo**
```json
http://{{host}}:{{port}}/wallet/accountpermissionupdate


{
  "owner_address": "41ffa9466d5bf6bb6b7e4ab6ef2b1cb9f1f41f9700",
  "owner": {
    "type": 0,
    "permission_name": "owner",
    "threshold": 2,
    "keys": [{
        "address": "41F08012B4881C320EB40B80F1228731898824E09D",
        "weight": 1
      },
      {
        "address": "41DF309FEF25B311E7895562BD9E11AAB2A58816D2",
        "weight": 1
      },
      {
        "address": "41BB7322198D273E39B940A5A4C955CB7199A0CDEE",
        "weight": 1
      }
    ]
  },
  "witness": {
      "type": 1,
      "permission_name": "witness",
      "threshold": 1,
      "keys": [{
          "address": "41F08012B4881C320EB40B80F1228731898824E09D",
          "weight": 1
        } 
      ]
    },
  "actives": [{
    "type": 2,
    "permission_name": "active0",
    "threshold": 3,
    "operations": "7fff1fc0037e0000000000000000000000000000000000000000000000000000",
    "keys": [{
        "address": "41F08012B4881C320EB40B80F1228731898824E09D",
        "weight": 1
      },
      {
        "address": "41DF309FEF25B311E7895562BD9E11AAB2A58816D2",
        "weight": 1
      },
      {
        "address": "41BB7322198D273E39B940A5A4C955CB7199A0CDEE",
        "weight": 1
      }
    ]
  }]
}

参数字段的定义及限制，请查看"2.1 结构说明"。

```

**active权限中operations的计算示例**

``` java
public static void main(String[] args) {

  //指定需要支持的合约id(查看proto中Transaction.ContractType定义)，这里包含除AccountPermissionUpdateContract（id=46）以外的所有合约  
  Integer[] contractId = {0, 1, 2, 3, 4, 5, 6, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 30, 31,
      32, 33, 41, 42, 43, 44, 45};
  List<Integer> list = new ArrayList<>(Arrays.asList(contractId)); 
  byte[] operations = new byte[32];
  list.forEach(e -> {
    operations[e / 8] |= (1 << e % 8);
  });
  
  //7fff1fc0033e0000000000000000000000000000000000000000000000000000
  System.out.println(ByteArray.toHexString(operations));
}
```


<h3 id="3.2">2）执行合约</h3> 

1、创建交易，与非多重签名交易的构建过程相同    
2、指定Permission_id，默认为0，表示owner-permission, [demo](https://github.com/tronprotocol/wallet-cli/commit/ff9122f2236f1ce19cbb9ba9f0494c8923a3d10c#diff-a63fa7484f62fe1d8fb27276c991a4e3R211)   
3、用户A签名，将签名后交易通过其他方式发送给B。  
4、用户B签名，将签名后交易通过其他方式发送给C。  
…  
n、最后一个完成签名的用户，将交易广播到节点。  
n+1、验证多重签名的权重之和大于域值则接受交易，否则拒绝交易  

代码示例：

[https://github.com/tronprotocol/wallet-cli/blob/multi_sign_V2/src/main/java/org/tron/demo/MultiSignDemo.java](https://github.com/tronprotocol/wallet-cli/blob/multi_sign_V2/src/main/java/org/tron/demo/MultiSignDemo.java)
  

<h3 id="3.3">3）其他新增接口</h3>

接口详细说明，请查看Tron-http.md与波场钱包RPC-API.md  
1、增加签名  
```text
curl -X POST  http://127.0.0.1:8090/wallet/addtransactionsign -d '{"transaction": "TransferContract", "privateKey": "permissionkey1"}'
 
rpc AddSign (TransactionSign) returns (TransactionExtention) {}
```
2、查询已签名地址
```text
curl -X POST  http://127.0.0.1:8090/wallet/getapprovedlist -d '{"transaction"}'
 
rpc GetTransactionApprovedList(Transaction) returns (TransactionApprovedList) { }
```

3、查询交易签名权重
```text
curl -X POST  http://127.0.0.1:8090/wallet/getsignweight -d '{"transaction"}'
 
rpc GetTransactionSignWeight (Transaction) returns (TransactionSignWeight) {}
```
  
## <h2 id="4">其他</h2>  

支持多重签名后，创建账户时有什么变化？  
在升级到V3.5版本后，并且多重签名提议生效后，创建账户时将自动生成owner-permission以及一个active-permission，
其中owner-permission中仅包含一个key，权限及阈值都为1。active-permission中也包含一个key，权限及阈值都为1，并且
operations为"7fff1fc0033e0000000000000000000000000000000000000000000000000000"，表示支持除
AccountPermissionUpdateContract以外的所有操作。
在V3.5版本以后，如果有新增系统合约，新创建的账户的默认operations值会发生变化。已经创建的账户的operations不会发生变化。








