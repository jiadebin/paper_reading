# Google论文阅读问题
------

## 1. GFS论文问题

1. 为什么存储三个副本？而不是两个或者四个？  
答：为了更好的容错性，其中两个位于同一个rack中，第三个位于其他rack，故障出现的情况可以分为：单机故障、rack故障，如果是单机故障那么rack内的第二个replica即可保证容错性，而如果rack故障，就需要位于其他rack的副本保证。

***扩展：********

对于Paxos这种多数派原则的一致性算法来说，四副本与三副本相比没有优势，因为它的多数派是3，最多也只能容忍1个副本错误（3个副本的多数派是2，也能容忍一个副本错误）。因此副本数通常选择奇数，比如要容忍f个副本故障，那么至少需要2f+1个副本。
三副本如何节省存储空间？两个副本中保存基线数据+更新日志，第三个副本只保存更新日志。

GFS集群并不是Paxos集群，对它来说，副本数越多，容错性越好。

2. Chunk的大小为何选择64MB？这个选择主要基于哪些考虑？  
答：64MB的chunk size相对较大，好处是：
a. 可以减少client与master的交互，因为读写一个chunk只需要请求一次master，chunk越大，可以处理的数据越多；
b. chunk越大，client能够在同一个与chunkserver的TCP连接来进行更多操作，如果chunk小了，操作相同量级的数据，则需要更多次TCP连接；
c. chunk越大，master需要维护的metadata越小，便于存储在内存中。

***扩展：*********
如果更大会有什么问题？越大越容易产生访问热点。
惰性空间分配依靠的是Linux文件系统机制，如果对于一个64M的文件，如果其数据没有达到64M，那么实际占用空间会小于64M。


3. GFS主要支持append，overwrite操作比较少。为什么这样设计？如何基于一个只支持Append操作的文件系统构建分布式表格系统Bigtable？  
答：如INTRODUCTION部分第三个设计点所述，文件的随机写操作在现实场景中基本不存在，一旦写入，通常只会有顺序读取的需求，不同类型的数据都具备这个特点。这种大文件的访问模式使得追加写重围GFS性能优化和原子性保证的关注点，同时client端缓存数据块也就失去了意义。

第二个问题：
将历史数据以文件形式保存在GFS中（SSTable），最新修改/写入数据保存在内存中（Memtable），log会从内存批量写入GFS。read操作merger历史数据和内存数据得到结果；write操作先写commit log，然后写memtable。
MinorCpmpaction：memtable达到阈值时生成一个SSTable文件写入GFS。这样可以避免server宕机重启后需要replay很多的commit log才能恢复memtable，减少restore的工作量。
MajorCompaction：对SSTable进行合并，回收deleted data所占空间。


4. 为什么要将数据流和控制流分开？如果不分开，如何实现Append流程？  
答：分开可以更有效地利用网络带宽。控制流从client流向primary chunk，然后再从primary流向secondary chunk，数据流从client流向最近的chunk server，再以流水线的方式流向其他chunk server，这种方式比树形传输数据更高效，每个server将接收的数据立刻传输给最近的邻居，全双工的网络链路保证发送数据的同时不会降低本机的接收速率，没有网络拥塞的前提下，传输B个字节给R个副本的时间为：B/T+R*L。其中T为网络带宽带宽，L为两个机器之间的传输延迟。
如果不分开，那么控制指令和数据应该一起传输给primary，primary确定指令执行顺序之后，将指令和数据一起分发给各个secondary，各个secondary执行完成回复消息给primary。


5. GFS有时会出现重复记录或者padding，为什么？  
答：两种情况导致padding：
（1）多个replica追加数据时，部分节点append失败，部分成功（或只追加了一部分），此时需要client重试，重试之前要把各个replica通过写padding形式进行末尾对齐；
（2）单次写入数据较多时，primary replica首先会判断会不会超过自己的boundary，如果会，则在当前chunk末尾填充padding至满状态，然后新建chunk进行数据追加，同时告诉其他replica也这么做。


6. Lease是什么？在GFS起什么作用？它与heartbeat有何区别？  
答：lease是租约机制，master赋予某个chunk的replica，它将作为该chunk的primary，然后client对该chunk的写操作都会去连接这个primary replica，无需再与master交互，primary也会发送有序指令给secondary replica。
lease的作用是减轻master的压力。
lease有超时机制，过期自动失效，也可被master强制revoke。
heartbeat是心跳信息，master定期与所有的chunkserver exchange heartbeat message，用来下发instruction和收集chunkserver的state。
heartbeat message中包含：chunkserver延长lease的请求、chunkserver所持有的chunk信息（master会回复它metadata中已经删除的chunk）。



7. GFS append过程中如果Secondary出现故障，如何处理？如果Primary出现故障，如何处理？  
答：P3.1节的write流程，大致是primary首先确定mutation操作顺序，然后自己本地执行，执行成功后再下发给各个replicas。
	append过程中secondary故障，primary会告诉client部分replica写入错误，client首先会从传输数据步骤开始重试，重试一定次数之后如果依然失败，那么client会从头重新走write流程联系master，master检查chunkserver状态，如果故障了，复制目标chunk到新的chunkserver上，然后告诉client持有lease的primary和chunk分布情况，client继续进行重试。
	这个过程可以优化：
	如果写入某个chunk时，两个副本写成功，第三个写失败，如果master协调复制出一份再继续写的话可能会很慢，复制过程比较耗时，优化方式是不再使用这个chunk，由于是append操作，不是overwrite，因此可以直接新开一个chunk2，chunk2又有三个副本，直接将数据写入chunk2的三个副本即可，而原先的chunk末尾加padding即可，这样效率更高。


（*******会有revoke lease吗？提升版本号，防止该节点再次上线时无法区分。论文中写到从第三个步骤传输数据开始重试，这意味着master应该不会revoke lease，GFS内部应该有一个机制去判断哪些chunk执行成功，哪些失败，哪个chunkserver需要重试。比如使用log去判断）

如果primary出现故障，那么client会得不到回应，超时后重试，重试一定次数后联系master，master会重新选择一个primary（等待lease超时，防止出现两个primary），并进行副本复制，然后将新的chunk primary信息发送给client，client重新执行append。


*8. GFS Master需要存储哪些信息？Master数据结构如何设计？  
答：目录/文件namespace、chunk的namespace、file-chunk的映射、chunk locations。
****namespace采用lookup table（将pathname映射到metadata，采用前缀压缩，可以放到内存中），可以用类似于B-Tree的结构实现，保证有序查找很快；
	file-chunks映射采用mutil_hashmap（一对多）；
	chunk-location信息采用mutil_hashmap（一对多）。

	checkpoint文件是压缩的B-tree结构。

9. 假设服务一千万个文件，每个文件1GB，Master中存储的元数据大概占用多少内存？  
答：1GB/64MB=16个chunk，每个chunk有64 Bytes的元数据，因此总共为10000000*16*64Bytes=10GB （每个chunk的handle是64bit，但是chunk的metadata是64B）
	每个文件的namespace元数据信息占用不超过64Bytes。


10. Master如何实现高可用性？负载的影响因素有哪些？如何计算一台机器的load值？  
* namespace、文件-chunk映射信息会持久化到磁盘，并存储在多个节点。
* 定期对metadata做checkpoint，checkpoint可以直接映射到内存，无解析开销，checkpoint也可以保证replay过程开销较小。
* master故障时，外部监控服务会让备机上线，提供读服务，备机会将master的日志replay，并监控chunkserver状态。

******扩展：**********
copy-on-write需要了解。

master负载影响因素：client读写请求量、re-replication、新建chunk。

load值：磁盘使用率、内存占用、CPU、网络带宽

11. Master新建chunk时如何选择ChunkServer？如果新机器上线，load值特别低，是否需要有些特殊考虑？  
答：P8、  
* 磁盘空间利用率低于平均利用率
* 限制每台chunkserver最近创建的chunk数量，因为新建chunk之后通常会有写数据请求
* chunk副本需要分布在不同rack上

需要限制每个节点的clone操作数量和clone操作占用的带宽，避免给该机器瞬间带来超大负载。

12. 如果某台ChunkServer报废，GFS如何处理？  
答：master通过heartbeat感知到该机器下线，对于存储在该机器上的chunk副本，选定其他机器进行clone操作，如果该机上有primary chunk角色，则为这些chunk重新选择primary。需要等待lease过期。


13. 如果ChunkServer下线后过一会重新上线，GFS如何处理？  
答：如果这段时间间隔内re-replication还没有执行，那么该机器重新上线后可能出现chunk版本落后的问题，client进行数据读取或者该机器本身进行扫描过程中会将版本信息汇报给master，master会发指令让该server从其他机器复制最新chunk，并将stale chunk标记为垃圾，如果version number是最新的，则该chunk可以继续上线。

14. 如何实现分布式文件系统的快照操作？  
答：流程如下：  
1. client向master发送快照请求
2. master收到请求后，revoke所有相关chunk的lease
3. 上述相关的lease过期后，master写入一条日志，然后再内存中复制一份快照涉及到的元数据  

快照操作完成后，client如果要向其中某个chunk写数据，就会请求master该chunk的位置信息（primary和secondary），然后master发现该chunk引用计数大于1（快照和原始metadata都在引用该chunk），它就向所有的chunk server发送复制该chunk为一个新chunk的请求，复制完成后，client将数据写入新的chunk，这是写时复制机制，旧的chunk就作为快照数据存储了。

checkpoint与snapshot区别：checkpoint一般用于故障恢复，且新版本一旦生成，老版本就可以删除了；而snapshot通常是数据在某个时刻的一个快照，可以作为版本数据来使用。


15. ChunkServer数据结构如何设计？  
答：要存储每个chunk包含的block（64KB）信息，block可以用chunk id + offset标识，每个block还有对应的checksum信息，用hashmap存储即可。
hashmap的key是chunk handle，value是其他信息。

16. 磁盘可能出现“位翻转”错误，ChunkServer如何应对？  
答：P10,5.2节。用checksum进行数据校验。  
* 读数据时，要检查所有block的checksum信息，确保数据没错误，如果错了就联系master进行正确的chunk复制。
* 写数据时，对于append操作，可以不检查checksum，等到读数据时再检查，因为append是在chunk末尾block追加数据，如果该block还没写满，那么后面再写入数据时，checksum是增量更新的，即使出错也可以在读取该block时检查出来；对于overwriter操作，需要首先检查要覆盖的range的第一个和最后一个block的checksum（只检查首尾block吗？论文里是这么写的），校验通过才进行写入，否则说明数据本身有错误，不应该被覆盖。

17. ChunkServer重启后可能有一些过期的chunk，Master如何能够发现？  
答：chunk server重启后，会向master汇报其存储的所有chunk和version number信息，master根据version可以判断其是否过期。如果过期了，master会让该server重新复制新的chunk，并不让这些chunk被client读取，将其删除。
如果master遇到了比自己version number大的chunk出现在某个chunkserver上，那么它会将该chunk的version number更新为这个更大的值。


讨论：
1、chunk的version number信息汇报一般不走heartbeat，heartbeat包含的信息通常很小。chunkserver重启之后汇报chunk信息的过程叫做report。

2、如果写入某个chunk时，两个副本写成功，第三个写失败，如果master协调复制出一份再继续写的话可能会很慢，复制过程比较耗时，优化方式是不再使用这个chunk，由于是append操作，不是overwrite，因此可以直接新开一个chunk2，chunk2又有三个副本，直接将数据写入chunk2的三个副本即可，而原先的chunk末尾加padding即可，这样效率更高。

3、GFS副本的特点是，三个副本必须同时在线且都写入成功才能返回成功，写入期间如果有副本宕机，则需要master协调生成新副本，这与Paxos的多数派协议是不同的，好处是三个副本可以同时提供读服务，而Paxos只有leader副本保证是最新的，多数派成员是不固定的。GFS二代产品开始采用Paxos一致性协议了。
