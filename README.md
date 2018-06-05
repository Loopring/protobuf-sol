## pb3sol
- protobuf3 solidity code generator, inpired by https://github.com/shmookey/solpb (only supports pb2) enhanced with:
  - [soltype-pb](https://www.npmjs.com/package/soltype-pb), which enhances inter-operability with [protobuf.js](https://github.com/dcodeIO/ProtoBuf.js/)
  - support native solidity types like address, uint256, by including special protobuf definition [Solidity.proto](https://github.com/umegaya/pb3sol/blob/master/src/protoc/include/Solidity.proto)
- unimplemented protobuf feature
  - enum. because solidity enum seems to not accept numerical value. 
  - float/double. no floating type in solidity. 



### Motivation
- recently, some kind of dapp like game need to treat complex, unstable data structure. 
- but for now, only solidity code can describe data structure and solidity code is immutable if it once deployed. 
 - actually, code can be replaced by [zos](https://zeppelinos.org/) or similar technology, but data related with old contract cannot move to new one automatically
- thus migrate data schema in smart contract is often with full database copy. its *un-acceptable* from the view point of performance and cost. 
- storing data as bytes and parse with protobuf could solve this problem with small increase of gas comsumption for each contract call, by separating actual data storage and its schema definition (replacable as solidity library)
- also, if we describe data as protobuf encoded byte array, contract only need to return byte array and parsing can be done on each client, which is huge save for contract code size and execution fee



### Try it out
- try to run test first. 
- easiest way to run test is using docker. clone this repository and run ```make tsh``` to enter shell, then ```cd test && make test_setup test_on_host```
- it does:
  - launch docker container and enter shell
  - install necessary node module by using npm install
  - compile proto files to solidity source
  - run [truffle](http://truffleframework.com/) test
    - [create storage contract](https://github.com/umegaya/pb3sol/blob/master/test/contracts/libs/Storage.sol) (contract which just holds byte array with string key)
    - [store byte array encoded by protobuf](https://github.com/umegaya/pb3sol/blob/master/test/contracts/Version1.sol#L14)
    - [load byte array and decoded by protobuf](https://github.com/umegaya/pb3sol/blob/master/test/contracts/Version1.sol#L48)
    - [check encoded value is correctly restored](https://github.com/umegaya/pb3sol/blob/master/test/contracts/Version1.sol#L53)
    - [load byte array and decode on js side](https://github.com/umegaya/pb3sol/blob/master/test/test/v1_access.js#L76)
    - [decode it with new version of proto file and its correctly migreated](https://github.com/umegaya/pb3sol/blob/master/test/test/v1_access.js#L99)



### Caveat with solidity <= 0.4.24
- at the time I wrote this (2018/03/03), latest released solidity version is 0.4.24, which cannot allow us to:
  - pass/return arbiter struct value to/from non-internal contract call
- so still we cannot use generated library for linking. 

### Caveat with solidity <= 0.4.20
- at the time I wrote this (2018/03/03), latest released solidity version is 0.4.20, which cannot allow us to:
  - return bytes from non-internal contract call
  - pass arbiter struct value to non-internal contract call
- as a result, read and parse protobuf encoded value from storage contract will be more expensive than it should be, because we cannot avoid do like [this](https://github.com/umegaya/pb3sol/blob/master/test/contracts/libs/StorageAccessor.sol#L19) to read arbiter length bytes from external contract for now.
- solidity authors [say](https://github.com/ethereum/solidity/pull/3308) this problem will solved with next solidity release (0.4.21?). 
- also this problem prevent our contract from linking with library generated by pb3sol, because we have to declare internal modifier for all encoder/decoder, which seems to prevent solc from generating delegatecall instruction.



### Basic Usage
- simplest way is using docker. for example, under truffle project do like following.
```
mkdir -p `pwd`/contracts/libs/pb 		# output directory for pb3sol
mkdir -p `pwd`/proto 					# input directory (proto files)
touch /proto/test.proto 				# create protofile (and edit) 

# compile test.proto as test_pb.sol with dependent library, runtime.sol and put into `pwd`/contracts/libs/pb 
docker run --rm -ti -v `pwd`/contracts/libs/pb:/out -v `pwd`/proto:/in umegaya/pb3sol protoc -I/ -I/protoc/include --plugin=protoc-gen-sol=gen_runtime=runtime.sol:/out /in/test.proto
```

- for actually usage in solidity codes, see [here](https://github.com/umegaya/pb3sol/blob/master/test/contracts/Version1.sol) and [here](https://github.com/umegaya/pb3sol/blob/master/test/proto/TaskList.proto#L24)



### Handling native solidity type
- if you use docker image, built-in proto file "[Solidity.proto](https://github.com/umegaya/pb3sol/blob/master/src/protoc/include/Solidity.proto)" can import straight forward like ```import "Solidity.proto"```. then you can use ```.solidity.$typename``` to declare variable which directly convert to $typename variable in solidity codes. 
  - if you don't want to use docker, copy above file into somewhere in your proto compiler $path. 

- basically these solidity type is bytes variable boxed with message. but convert these bytes into correct number or bigint, is not trivial work. 
  - we create node module [soltype-pb](https://www.npmjs.com/package/soltype-pb) to add small support for handling these native solidity types with protobufjs.
  - installation to your project: ```npm install soltype-pb```
  - see [here](https://github.com/umegaya/pb3sol/blob/master/test/test/v1_access.js#L57) and [here](https://github.com/umegaya/pb3sol/blob/master/test/test/v1_access.js#L6) for usage.

