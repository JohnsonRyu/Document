
# 1. Introduction

This project is to implement a compensation scheme on Klaytn blockchain platform to record data and provide rewards to 'Inconvenience Box' application operated by Nitpick([https://www.nitpick.co.kr/](https://www.nitpick.co.kr/). Users receive two kinds of rewards, Social Innovator Token and Point, after writing their inconvenience experiences in the app. Social Innovator Token is the means of purchasing the right to raise the level and convert the user's nickname within the app, and the point is the means of purchasing various items at the Store. The higher the level of a user, the more points the user can collect and the more items they can buy at the Store. In the future, it is possible to provide multiple rewards to users by tracking where and how the users' inconvenience experiences used.
   
# 2. Systems Architecture

![image_1](./images/image_1.png)

## API Function List

* User registration 
* Retrieve all users
* Token request
* Token transfer
* Check a balance
* Upload an Inconvenience post
* Retrieve an Inconvenience post
* Retrieve all Inconvenience post
* Level up for a user
* Check for user's level
* Purchase a ticket for changing a user's nickname.
* Change a user's nickname.
* Check a user's status
    
The Sender of all Transactions created on Klaytn API Server is User itself. In the case of Public Blockchain such as Ethereum, the Sender should pay Transaction Fee, but in Klaytn, the Transaction Fee can be paid by others depending on situations. We create a master account which is hosted by Fee Delegated, deposit enough Klay there, and let the master account pay Transaction Fee instead of the user.
   
- [Enterprise Proxy](https://docs.klaytn.com/klaytn/enterprise_proxy#necessity-of-enterprise-proxy)
- [Klaytn Send Transaction Fee Delegated](https://docs.klaytn.com/api/toolkit/caverjs/caver.klay#sendtransaction-value_transfer)
   
   
# 3. SmartContract Code OverView

 This pilot project has total 3 kinds of SmartContracts.
   
![image_2](./images/image_2.png)
   
* `Inconvenience` Contract uploads and saves posts.

* `Social Innovator Token` Contract has the function of a user's reward and means of payment. 
    - We use ERC20 to create a token that serves as a means of payment.
    - Check out detailed information about ERC20 at [eips.ethereum.org](https://eips.ethereum.org/EIPS/eip-20)

* `InconvenienceNFT` Contract which is non-fungible token contains nickname and level information of a user.
   - We use ERC721 standard to create a non-fungible token.  
   - Check out detailed information about ERC721 at [erc721.org](erc721.org)
   
## 3-1. SIToken (Social Innovator Token)
> Name : Social Innovator Token  
Symbol : SIT  
Decimals : 18  

   
We use ERC20 standard, also add `Pausable` contract to prepare for special situations, `BlackList` contract to prevent fraudulent users, and `MultiTransferToken` contract to transfer the token to multiple addresses and it can be allowed only for administrators.


## 3-2. Inconvenience
   
Inconvenience Contract uploads user's posts and offers rewards.

Uploading a post is simply recording title and content of the post. However, retrieving posts and offering rewards according to specific conditions require several additional data. 
   
Therefore, we separate those functions into 3 different contracts, which are RewardPool, UserStore, and InconvenienceStore.

Each contract does not call each other and those contracts manage Reward, UserData, InconvenienceData respectively.

### 3-2-1. 'InconvenienceStore' Data Structure
   
InconvenienceStore Contract is a contract that saves a `Inconveience` post as a struct type.  

Each of the posts is recorded in the ‘n’th index of an array. 
If the post is successfully uploaded, information recorded in the ‘n’th index of the array will be notified to the owner who wrote the post.
   
In `Inconvenience` struct type, there are `owner` who uploaded the post, `id` that is the unique number of the post recorded on the Web server, `tag` that is title of the post, `content` that is content of the post, and `timestamp` that indicates the time when the post is recorded on the blockchain.

Uploaded posts are saved in an array of the struct.
   
```
event insertInconvenienceEvent (uint256 _idx, address _address, string _id, string _tag, uint256 _timestamp);
    
struct Inconvenience {
    address owner;
    string id;
    string tag;
    string contents;
    uint256 timeStamp;
}

Inconvenience[] private inconveniences;
```
   
   
### 3-2-2. 'UserStore' Data Structure
   
UserStore Contract maps User Data to User's Public Address and then save it.

There is a reason why we save User as mapping type and uploaded post as an array type.
If we save user as an array type, we have to pay more GAS to check for the duplicate users.
Therefore, saving users in mapping type rather than array type is more effective.

In case of the uploaded post, users have Index of their own posts in their arrays.
Therefore, using array data type rather than mapping can save GAS.

In `UserData`, there are `userCount` indicating registration number on blockchain, `rewardCount` that can check amounts of daily rewards, `lastTimeStamp` which is for initializing amounts of daily rewards, and `userInconv` that saves a user's post as a number and use it to retrieve the user-owned pos.
   
```
event NewMemberEvent(address _user, uint256 _userCount, uint256 _timestamp);

struct UserData {
    uint64 userCount;
    uint8 rewardCount;
    uint256 lastTimeStamp;
    uint256[] userInconv;
}
    
mapping(address => UserData) private userStore;
```
   
   
### 3-2-3. SignUp function
   
```
function signUp() external canSignUp
```
   
In Inconvenience Contract, Sign Up process is required before uploading posts. 

Because, `userCount` which represents ‘n’th user in UserData should be increased when signUp is finished, not uploading the post.
   
### 3-2-4. Post Inconvenience function
   
```
function postInconvenience(string _id, string _contents, string  _tag) external onlyMember {
    insertUserInconv(inconvCount);        
    insertInconvenience(_id, _contents, _tag);
    checkUserRewardCount();
}
```
   
Uploading posts is only available to registered users.
This restriction is not to prevent users from accessing through the contracts in an inappropriate way without accessing through the service. SignUp function can be called by anyone even if they do not access through the service. 
The reason for checking registered users is to verify whether `totalUserCount` was increased through the normal signUp process or not. 
   


`insertUserInconv(inconvCount);` function stores the index of a new post in the user's UserData. The user can have indexes of their posts as an array through this function.
   
`insertInconvenience(_id, _contents, _tag);` function saves a post delivered from Klaytn API in an array.
   
```
function insertInconvenience(string _id, string _contents, string  _tag) internal {
    Inconvenience memory _inconvenience = Inconvenience({
        owner: msg.sender, 
        id: _id,
        contents: _contents,
        tag: _tag,
        timeStamp: now
    });

    inconveniences.push(_inconvenience);        
    inconvCount = inconvCount.add(1);

    emit insertInconvenienceEvent(inconvCount, msg.sender, _id, _tag, now);
}
```
   
   
The newly generated post is pushed to `inconveniences` array as above. 
When we assume that retrieving `a post uploaded by A User`, we can iterate through by looking at the inconveniences.length property to check if the owner of the post is A or not. 
(Retrieving datas in Klaytn Network does not spend Gas)
However, this will increase the cost of searching whenever the posts are increased. This is the reason why we store the Index of the user-generated post in UserData.

   
`checkUserRewardCount();` performs a function of `checking counts of daily rewards` and `offering rewards`.
 
```
function checkUserRewardCount() internal {
    checkResetRewardCount();
    
    if(userStore[msg.sender].rewardCount < dailyRewardCount) {
        tokenTransfer();
        IncreaseRewardCount();
    }
    
    changeLastTimeStamp();
}
```
   
Firstly, `checkResetRewardCount();` function checks initialization of the counts of rewards which is received by the User. timestamp should be converted to date because the counts of rewards have to be initialized every 00 hour based on UTC+09. We use DateTime Contract for this function. 

- Check out detailed information about DateTime at [github](https://github.com/pipermerriam/ethereum-datetime)
   

In addition, for efficient use of GAS, `UserData.lastTimeStamp` recorded by the user is converted to Day and current TimeStamp is converted to Day as well. Then we compare these two Days. If these two Days are same, we check two Months additionally. In the case of Years, the cost of GAS spent by the conditional testing is estimated to be higher than the possibility that they would be totally the same, so we do not check the years.



```
function checkResetRewardCount() private {
    // UTC+09:00
    uint8 lastDay = getDay(userStore[msg.sender].lastTimeStamp + 32400);
    uint8 curDay = getDay(now + 32400);
    
    if(lastDay != curDay) {
        userStore[msg.sender].rewardCount = 0;
    } else {
        uint8 lastMonth = getMonth(userStore[msg.sender].lastTimeStamp + 32400);
        uint8 curMonth = getMonth(now + 32400);
        
        if(lastMonth != curMonth) {
            userStore[msg.sender].rewardCount = 0;
        }
    }
}
```
   
Secondly, we check a number of daily rewards. If the user is eligible for getting rewards, we offer user’s rewards with `tokenTransfer();` function. After that, we increase the counts of daily rewards through `IncreaseRewardCount();` function.


Lastly, we update `UserData.lastTimeStamp` by using `changeLastTimeStamp();` function to record the lastest rewards time of the user. This is for initializing the counts of daily rewards referred in the first step. 
   
## 3-3. InconvenienceNFT
> Name : Inconvenience Rank  
Symbol : IR
   
We follow ERC721 standards, and the user's NFT includes the user's nickname and level, and users can increase their level and change the nickname if they pay a certain SIT.
   
   
### 3-3-1. 'InconvenienceNFT' Data Structure
   
TokenData includes User's Level, NickName and purchase situation of a ticket for changing NickName. 
TokenData is mapped to the index number of tokens, and the costs to be paid for each level up is saved in a dynamic array for future updates. 
   
   
```
struct TokenData {
    uint256 level;
    string nickName;
    bool nickNameTicket;
}
mapping (uint256 => TokenData) private tokenDataList;
uint256[] private LvUpCost;
uint256 private nickNameCost;
```
   
   
### 3-3-2. Mint Unique Token Function
   
```
modifier canMint() { require(balanceOf(msg.sender) == 0); _; }

function mintUniqueToken(string nickName) external canMint {
    uint256 tokenId = totalSupply();
    _mint(msg.sender, tokenId);
    TokenData memory newTokenData = TokenData({
        level : 0,
        nickName : nickName,
        nickNameTicket : false
    });
    tokenDataList[tokenId] = newTokenData;
    emit TokenMint(msg.sender, tokenDataList[tokenId].level, tokenDataList[tokenId].nickName);
}
```
   
NFT we are using is a tokenized version of the user's activities, so there is no need for a single user to own multiple tokens, nor for tokens to be transferred to someone. Therefore, we limit users to create one token for each user through `canMint()` function. 
   
### 3-3-3. Level Up & Change NickName
   
Level Up function and purchase ticket function for changing NickName operate with the same processes in below. 

(1) Is it a condition for users to make a purchase?   
(2) Does the user allow Social Innovator Token as much as the costs of the item to purchase?   
(3) TransferFrom the user's SIT to the administrator's account.    
(4) Update purchased items.   

# 4. Improvements
   
   
### 4-1. Gas Price
   
In Smart Contracts, such as `3-2-3`Sign Up, certain data causes the problem of adding conditional statements or separating functions. If userCount does not exist in UserData in this situation, the number of functions could be decreased, and the conditions checked in the function which is called by users each time when they upload posts could be decreased, thereby the Gas Used by Transaction could be reduced. We have a necessity to compare the Gas that can be reduced by without using a `userCount` variable and the need for `userCount`.
   
### 4-2. Prevention from abusing
   
All transactions such as signing Up, uploading posts are signed with the user's Private Key. It means the user can call functions of the Contract directly, also user can access them through the service.
However, since all Transactions that occur in the service operate as Fee Delegated, there is no burden on users about paying Transaction Fee. On the other hand, direct access from the Contract makes the users pay Transaction Fee directly, so they have no benefits. 

However, if the value of the token they can get from posting is higher than the Transaction Fee that they pay for uploading posts, they will make abusing regardless of the Transaction Fee. The easiest way to resolve this problem is to sign the Transaction with the Administrator's Private Key rather than with the User's Private Key. However, this method does not take advantage of Fee Delegated function of Klaytn, and it can mean losing 'decentralization'.

* [First Method] As mentioned above, the administrator signs the Transaction.
    - If the function is accessible only by the administrator, it is not possible for general users to access to the Contract.
    - It is a simple method can be used in the Public Blockchain, so using Klaytn Blockchain has no benefits.

* [Second Method] Separate uploading post function & offering rewards function and add validation logic for the posts.
    - Currently, rewards are offered immediately at the same time as uploading posts. We need to check if the posts are worth the rewards.
   
### 4-3. Improvement of uploading posts methods.

When we uploaded posts at first, we saved them as String without any encryption process. However, estimating the user's Gas price was difficult. Also, encryption was needed in systems such as 'Secret Posts' that only certain people could see. AES(Advanced Encryption Standard) and SHA-256(Secure Hash Algorithms) encryption methods were considered.

AES increases the length of encrypted data more than original data but can be decrypted. 
SHA-256 cannot be decrypted, so data recorded in blockchain should only be used for `verification`. However, it has advantages that the Gas price is reduced exceedingly and we are able to predict the Gas price of Users. 

Although Klaytn Network is known to have lower Gas price than the Public Blockchain such as Ethereum, however, storing original data on the blockchain is not a good way for the Klaytn Network, either in terms of the Gas price. That is the reason why we choose SHA-256 encryption. But if we post the decryptable data, it is possible to sell 'Inconvenience data' as methods of second and third sales through P2P.
   
### 4-4. 'Like' function.
   
Currently, users get rewards ‘n’ times a day without any strings attached when they post. But except posting, rewards for 'like' is being processed by existing Web Server. It was one of the functions that we gave up because it is a pilot project and we had a time limitation. We can resolve this problem by adding data about 'like' to UserData and Inconvenience Data. However, we need to consider about Gas since it requires more strict conditional testing than offering posting rewards. 

   
### 4-5. Separation of Business Logic from Storage
   
Regarding this project, we are sorry that we did not make a separation of Contract and Storage.
If we update this project or change the way it operates, the User Data and Inconvenience Data which are recorded during the service period cannot be used. This is one of a drawback of SmartContract. It may be impossible to add Data even if we separate Contract and Storage. However, if we separate Storage and Logic, we could prepare for certain changes at least.
