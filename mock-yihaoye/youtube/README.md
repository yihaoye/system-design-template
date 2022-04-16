# Youtube 系统设计

[花花酱 Youtube 系统设计](https://www.youtube.com/watch?v=mp-OSK6jm1c)  
[Video Streaming System Design](https://medium.com/double-pointer/system-design-interview-video-streaming-service-e-g-netflix-or-youtube-design-adc2402e54a1)  
[Design Youtube](https://systeminterview.com/design-youtube.php)  
  
主要步骤：  
1. 明确需求（功能性与非功能性）- 面试时间有限，所以挑 2-3 个重点功能进行设计（需要先和面试官达成一致）
   1. Feature 需求（功能性需求）
      1. 上传视频（重点功能）
      2. 观看/播放视频（重点功能）
      3. 搜索（一般重点功能）
      4. 分享
      5. Like/Dislike
      6. 评论
      7. 推荐等等...
   2. 流量/用户量（日活用户）
      1. Consistency（一致性）
         1. 得到的数据永远是最新的
         2. Tradeoff with Availability: Eventual Consistency（最终一致性）
      2. Availability（高可用）
         1. 系统流量比较大的时候的扩展性与低延迟性能（重点非功能）
         2. 每个请求可得非报错响应，即不严格要求一定返回的是最新写数据（最终一致性）
      3. Fault Tolerance（系统容错性）
         1. 即使网络节点的原因导致随机的（arbitrary）数据丢失或延迟，依然不影响系统正常运转
2. 带宽存储估算
   * 带宽存储估计分析在实践中也需要，当一个新项目开始前，需要根据此类分析估算基建资源的需求（基建资源一般要提前几个月时间申请）
   * 了解系统大致规模，从而采用相应的设计方案（扩展性相关）
   * 2 Billion（20 亿）总用户量，1.5M 日活用户（DAU）
   * 上传用户（假设占总用户的 1%）上传发布视频的频率为 1 video/week
   * 观看用户（DAU）每天平均看 50 min
   * 视频长度，假设每个 10 min、分辨率 1080p（假设观看此分辨率的带宽是 5Mbps）
   * 存储估算
     * 新视频 - 2B * 1% / 7 days = 33 videos/sec（33 videos/sec 指的是每秒有 33 个视频被完整上传）
     * 每分钟文件大小 - 1080p 视频要转码到低分辨率（720p、480p、360p）以满足不同的设备与带宽需求，即 100 MB/min
     * 每天新增存储 - 33 videos/s * 86400s * 10min (视频长度) * 100 MB/min = 2.7 PB/day
     * 数据的备份 - 冗余：一个 region（数据中心）做 3 个备份；可用性：跨 region（数据中心）做 3 个备份；总共是 9 倍
   * 带宽估算
     * 上传带宽 - 33 videos/sec * (10 * 60s) * 5Mbps = 99Gbps - 这里使用 5Mbps 而不是上面的数据是因为此时不用考虑分辨率转换，10 是视频均长为 10 分钟，60 是一分钟有 60秒
     * 下载/观看带宽 - 同时用户数量： 150M DAU * 50mins / (24 * 60 mins/day) = 5.2M 个用户；带宽：5.2M * 5Mbps = 26Tbps；读写比例：26Tbps / 99Gbps = 263 : 1（即读 heavy 系统 - 决定后续 High Level 设计与扩展）
3. System APIs（可以使用 SOAP 或 REST API）
   * uploadVideo(string apiToken, string videoTitle, string videDesc, stream videoContents) - 因为视频的处理需要比较长的时间，所以服务器会采用异步处理，用户此时去做其他事情没必要傻等，处理完系统平台会通过邮件或平台 UI 上通知上传完毕。上传过程细节如下：
     1. 客户端发起 HTTP 请求以建立信道，此时服务端开始：a. 创建 videoID、b. 准备存储空间位置（上传目标地址如 S3）、c. 等等
     2. 服务端返回上传目标地址，客户端开始读取 video 并分块上传到对应的地址，Youtube 实际上采用 HTTP 协议来分块上传视频文件
   * streamVideo(string apiToken, string videoId, int offset, int codec, int resolution) - codec 是视频的编码格式（与用户设备是否支持有关），resolution（分辨率）可以自动根据当时带宽大小决定以优化观影体验（参考：[基于 HTTP 的动态自适应流](https://zh.wikipedia.org/wiki/%E5%9F%BA%E4%BA%8EHTTP%E7%9A%84%E5%8A%A8%E6%80%81%E8%87%AA%E9%80%82%E5%BA%94%E6%B5%81)）。该 API 将返回一个已经位移了 offset（时间戳）的视频流，使用的流媒体协议可以是 RTMP 之类的（关于更多流媒体协议的细节，可以看下面的单独章节）。
   * searchVideo(string apiToken, string searchKey, string userLocation, string pageToken)
4. High Level 整体系统设计（架构图）
   * 上传架构：视频转码比较耗时所以需要一个消息队列进行异步处理。过程如下：当视频上传后，Upload Service 会往队列里添加一个视频处理任务，下游会有一个 Video Processing Service 从队列里取任务，然后从文件系统（Distributed Media Storage - 比如 S3、HDFS、GlusterFS）下载相应视频进行处理（转码、提取缩略图等）完后把新的视频和缩略图存放到文件系统，同时在 metadata 数据库中更新视频与缩略图的存放地址。系统对用户播放延时要求比较高，所以会把视频 push 到离用户比较近的服务器（CDN），Video Distributed Service 就是负责把视频和图片分发到 CDN 各个节点上，因为比较耗时所以也是异步进行（从 Completion Queue 取任务） ![](./Upload%20Architecture.png)
     * Video Processing Service 从文件系统下载视频并分割成多个小块/小片段，然后并行同时地（更快）对它们进行解码然后再编码（变成不同的设备或文件格式和分辨率），除此之外还会进行缩略图获取以及机器学习对视频内容分析等等处理。（上传的视频不会存储为单个大文件，而是存储在多个小块中，因为可能会上传大视频，处理或流式传输单个大文件会非常耗时。并且当视频以块的形式存储后，当观看者使用时无需在播放之前下载整个视频，它将从服务器请求第一个块，当该块正在播放时，客户端请求下一个块，因此块之间的延迟最小，用户可以在观看视频时获得无缝体验）
       * 原始视频通常是一种高清格式，大小可以以 TB 为单位。Netflix 系统为支持多种设备和不同的网络速度，为每部电影或节目生成并存储了 1200 多种格式。需要将每个视频转换成不同的格式（mp4、avi、flv 等），以及每种格式的不同分辨率（例如 1080p、480p、240p 等），假设支持 i 种格式和 j 种分辨率，系统将为上传到平台的每个视频生成 i*j 个视频。这些块中的每一个的条目都将被写入元数据数据库，包括保存实际文件的分布式存储的路径。
       * 不同的内容创作者可能有不同的视频处理要求。例如，一些需要在视频添加水印，一些自己提供缩略图，一些上传高清视频，而另一些则不需要。为了支持不同的视频处理流水线并保持高并行性，可以添加某种抽象级别并让客户端程序定义要执行的任务。例如，Facebook 的流媒体视频引擎使用有向无环图 (DAG) 编程模型，该模型分阶段定义任务，以便它们可以按顺序或并行执行。视频转码过程可以采用类似的 DAG 模型来实现灵活性和并行性。
     * CDN 是内容分发系统（视频、图片是静态内容，非常适合 CDN 的分发），在不同地区都有服务器，CDN 通常重点使用缓存技术，通过缓存内存来服务绝大部分的内容。并不是所有视频都会分发到 CDN（CDN 容量也有限），比如冷门视频（每天只有 1-20 次观看量）会从原数据中心 stream 给用户（热门视频由 CDN stream 给用户）。
   * 下行（播放）架构：用户客户端播放请求通过 LB 转发给 Video Playback Service，该服务主要负责视频流的播放，与该服务协作还有一个 Host Identify Service（给定一个 video 和用户 IP 地址、设备信息，然后查找离用户最近的最适合设备与分辨率的视频的 CDN 地址，然后用户可以通过该 CDN 直接观看该视频了，如果没找到任何 CDN 地址则给出原数据中心地址），其他视频元数据如标题、描述等从 metadata 数据库中获取。![](./Watch%20Architecture.png)
   * 搜索架构
   * 另外需要注意的是实际中上面每个架构的每一个 Service（比如 Upload Service）可以是同一个微服务的集群，并不是单独一个机器、节点、应用。
5. 数据存储设计
   * 数据库表设计：
     * User: {VARCHAR(32) userID, VARCHAR(255) name, VARCHAR(255) email, BIGINT(20) numSubscribe...}
     * Video Metadata: {VARCHAR(32) videoID, VARCHAR(32) userID, VARCHAR(100) title, VARCHAR(255) desc, VARCHAR(255) videoAddr, VARCHAR(255) thumbnailAddr, BIGINT(20) numLike, BIGINT(20) numDislike, BIGINT(20) numView...} - 其中 numLike、numDislike、numView 应该拆分到另外的表或数据库，因为它们是大数据而且写频率非常高，在做这类数据的统计时，需要用到一些相关的数据结构，比如 [HyperLogLog](https://github.com/yihaoye/data-structure-and-algorithm-study-notes/blob/master/Common%20Algorithm%20and%20Theory/HyperLogLog.md)，[Redis 里有内置的 HyperLogLog 数据结构及相关工具供使用](https://mp.weixin.qq.com/s/MVxmH5r0tP6sQ9WKVLoDbw)
     * Comment: {VARCHAR(32) commentID, VARCHAR(32) userID, VARCHAR(32) videoID, VARCHAR(255) content, TIMESTAMP time...}
   * 数据存储选择：
     * SQL - 适合存储 User、Video Metadata 表
     * NoSQL - 适合非结构化数据，比如在 BigTable 里存储视频缩略图往往可以优化性能（不过实际上许多企业还是放在 File System 里）
     * File System / Blob Storage - 特别适合存储多媒体文件（图片、音频、视频等等）。因为视频平台的数据量很大，所以 File System 需要使用分布式的文件系统比如 HDFS、GlusterFS，又或者使用 Blob Storage / Object Storage 比如 Netflix 使用 AWS S3（S3 是 Object Storage，但也是通过类似 GFS2 之类的 Blob Storage 配合缓存、元数据存储等等组件、API 服务实现的，[参考](https://www.youtube.com/watch?v=9_vScxbIQLY)）。
6. 扩展性与优化
   * 找出系统的潜在瓶颈，然后提出解决方案、优缺点分析、然后做取舍
     * 数据分片 - 按 videoID 分片并且使用一致性哈希的方法。优势：按 videoID 而不是 userID 的好处是避免热门 up 主的问题。劣势：查询一个 user 的 videos 需要查询所有的 shards，以及单个热门视频会导致大流量集中在单个数据库（解决方案 - 使用缓存存储热门视频或拷贝视频到多个服务器上）。
     * 数据拷贝 - Primary-secondary 配置，数据写入 Primary 然后 propagate 到所有 secondaries 节点。从 secondary 读取。优势：可用性，随时可读不影响写压力。劣势：影响了数据一致性 - 只能是最终一致性，即用户不一定能读到最新数据（在本系统可接受这个取舍）。
     * 负载均衡 - 服务或者数据拷贝到多个机器，来服务大流量以提高系统可用性与降低延迟。
     * 自动扩展 - 当流量激增时，通过监控微服务的资源使用率（如 CPU 或内存）高于阀值时，增添或减少更多的服务副本。
     * 缓存 - Redis 以及 CDN 等。特别适合 read heavy 系统（视频系统读写比例是 200 : 1，是典型的 read heavy）。在任何读数据的地方都可以放缓存以减少对数据库的 hit，策略可使用最近最少使用（LRU）。
       * 需要缓存多少？采用二八原则，缓存 20% 每天读取的数据（daily read videos）。
       * 缓存会很大，如数据库一般也需要分布到多台机器上，分布机制可采用一致性哈希方法。
       * CDN 优化，通过分析预测，提前把客户会观看的视频推送到离用户最近的 CDN 上（可以在 off-peak 如半夜时段进行视频分发推送，降低单一时段的带宽压力）。
       * 通过和 ISP 合作进一步优化延时（Netflix Open Connect - OC Server/Appliance in ISP networks），把 Cache 放到 ISP 里，当用户请求某个视频且 ISP 发现它的 Cache 里有，就可以从它的 Cache stream 给用户，好处是相比 CDN（访问 CDN 仍需通过 ISP）少一步。实例：据报道，90% 的 Netflix 视频流量都是从 ISP 里的 Cache 走的。这种办法比较适合点播网站，即视频数量较少、观看比较集中。
     * 上传速度优化
       * 并行视频上传 - 将视频作为一个整体上传是低效的，可以通过 GOP alignment 将视频分割成更小的块，这允许在上一次上传失败时快速恢复上传。客户端可以实现通过 GOP 分割视频文件以提高上传速度。
       * 上传中心靠近用户 - 另一种提高上传速度的方法是在全球范围内建立多个上传中心，比如美国人可以上传视频到北美上传中心，中国人可以上传视频到亚洲上传中心。为此，系统使用 CDN 作为上传中心。
     * 消息队列
       * 另一个优化是构建一个松耦合的系统并启用高并行性。系统设计需要一些修改来实现高并行度。以下是视频从原始存储传输到 CDN 的原始流程，表明输出取决于上一步的输入，这种依赖性使并行性变得困难。为了使系统更松耦合，引入了消息队列，在引入消息队列之前，编码模块必须等待下载模块的输出，引入消息队列后，编码模块不再需要等待下载模块的输出，如果消息队列中有事件，编码模块可以并行执行这些作业。
     * 错误处理 - 构建一个高度容错的系统，可以高效处理错误并快速从错误中恢复。
       * 可恢复的错误。对于视频片段转码失败等可恢复的错误，一般的思路是重试几次操作。如果任务继续失败并且系统认为它不可恢复，它会向客户端返回相关的错误代码。
       * 不可恢复的错误。对于视频格式错误等不可恢复的错误，系统会停止与视频相关的正在运行的任务，并将相关的错误代码返回给客户端。
       * Typical errors for each system component are covered by the following playbook:
         * Upload error: retry a few times.
         * Split video error: if older versions of clients cannot split videos by GOP alignment, the entire video is passed to the server. The job of splitting videos is done on the server-side.
         * Transcoding error: retry
         * Preprocessor error: regenerate DAG diagram
         * DAG scheduler error: reschedule a task
         * Resource manager queue down: use a replica
         * Task worker down: retry the task on a new worker
         * API server down: API servers are stateless so requests will be directed to a different API server.
         * Metadata cache server down: data is replicated multiple times. If one node goes down, you can still access other nodes to fetch data. We can bring up a new cache server to replace the dead one.
         * Metadata DB server down: Master is down. If the master is down, promote one of the slaves to act as the new master. Slave is down. If a slave goes down, you can use another slave for reads and bring up another database server to replace the dead one.
  

**视频去重**  
拥有大量用户，上传大量视频数据，服务将不得不处理许多重复的视频。重复的视频通常在宽高比或编码方面有所不同，可能包含叠加层或额外的边框，或者可能是较长的原始视频的摘录。重复视频过多会在多个层面产生影响：  
* 数据存储：保留同一视频的多个副本会导致浪费存储空间。
* 缓存：重复的视频会占用可用于独特内容的空间，从而导致缓存效率下降。
* 网络使用：增加必须通过网络发送到网络内缓存系统的数据量。
* 能源消耗：更高的存储、低效的缓存和网络使用会导致能源浪费。
  
对于用户而言，这些情况将导致：重复搜索结果、更长的视频启动时间和视频流传输中断。  
对于系统而言，相比于在稍后再查找处理重复视频，在用户上传视频的早期就删除重复数据更有意义。内联重复数据删除将节省大量可用于编码、传输和存储视频副本的资源。一旦任何用户开始上传视频，服务就可以运行视频匹配算法（例如，[块匹配 | Block Matching Algorithm](https://en.wikipedia.org/wiki/Block-matching_algorithm)、[相位相关 | Phase Correlation](https://en.wikipedia.org/wiki/Phase_correlation)等）来查找重复项，如果发现已经有正在上传的视频的副本，可以停止上传并使用现有的副本，或者使用新上传的视频（如果它的质量更高）。如果新上传的视频是已有视频的子部分（反之亦然），可以智能地将视频分成更小的块，这样只上传那些不重复的部分。  
  
**视频流式传输协议**  
当用户在 Youtube 上观看视频时，通常视频流会立刻开始播放而不是等待整个视频下载好后再播放。下载整个视频意味着把视频复制到客户端，而流式传输（streaming）意味着客户端连续接收来自远程源视频的视频流，当用户观看流媒体视频时，客户端一次会加载一点数据，因此可以立即、连续地观看视频。  
在讨论视频流之前，先看一个重要的概念：流协议（streaming protocol）。这是一种控制视频流数据传输的标准化方法。流行的流媒体协议有：  
* MPEG–DASH：MPEG 代表 "Moving Picture Experts Group"，DASH 代表 "Dynamic Adaptive Streaming over HTTP"，中文名是**基于 HTTP 的动态自适应流**，它是一种自适应比特率流技术，也是一项国际标准，使高质量流媒体可以通过传统的 HTTP 网络服务器以互联网传递。类似苹果公司的 HTTP Live Streaming（HLS）方案，MPEG-DASH 会将内容分解成一系列小型的基于 HTTP 的文件片段，每个片段包含很短长度的可播放内容，而内容总长度可能长达数小时（例如电影或体育赛事直播）。内容将被制成多种比特率的备选片段，以提供多种比特率的版本供选用。当内容被 MPEG-DASH 客户端回放时，客户端将根据当前网络条件自动选择下载和播放哪一个备选方案。客户端将选择可及时下载的最高比特率片段进行播放，从而避免播放卡顿或重新缓冲事件。也因如此，MPEG-DASH 客户端可以无缝适应不断变化的网络条件并提供高质量的播放体验，拥有更少的卡顿与重新缓冲发生率。
* Apple HLS：HLS 代表 "HTTP Live Streaming"
* Microsoft Smooth Streaming
* Adobe HTTP Dynamic Streaming (HDS)
  
不需要完全理解甚至记住这些流协议名称，因为它们是需要特定领域知识的底层细节。重要的是要了解不同的流协议支持不同的视频编码和播放器。当设计视频流服务系统时，必须选择正确的流协议来支持面对的用例。[流媒体协议更多细节](https://archerzdip.github.io/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E7%B3%BB%E5%88%97-%E6%B5%81%E5%AA%92%E4%BD%93%E5%8D%8F%E8%AE%AE%E7%AF%87-%E5%B8%B8%E8%A7%81%E7%9A%84%E6%B5%81%E5%AA%92%E4%BD%93%E5%8D%8F%E8%AE%AE%E4%BB%8B%E7%BB%8D/)  
  
视频系统的其他拓展功能：  
* 推荐
* 搜索
* 直播（Live Stream）
* 收藏
  
[关于 Netflix](http://highscalability.com/blog/2017/12/11/netflix-what-happens-when-you-press-play.html)  
在用户点击播放之前发生的所有事情都发生在后端，该后端在 AWS 中运行。这包括准备所有新传入的视频和处理来自所有应用程序、网站、电视和其他设备的请求。（任何不涉及提供视频的内容都在 AWS 中处理。这包括可扩展计算 EC2、可扩展存储 S3、业务逻辑、可扩展分布式数据库 DynamoDB 和 Cassandra、大数据处理和分析、推荐、转码以及数百个其他功能。）  
在用户点击播放后发生的所有事情都由 Open Connect 处理。Open Connect 是 Netflix 的自定义全球内容交付网络 (CDN)。Open Connect 将 Netflix 视频存储在世界各地的不同位置。用户客户端的视频流主要来自 Open Connect（CDN）。  
Netflix 在三个 AWS 区域运营：一个在北弗吉尼亚州，一个在俄勒冈州波特兰市，一个在爱尔兰都柏林。在每个区域内，Netflix 在三个不同的可用区运营。拥有三个区域的好处是任何一个区域都可以发生故障，而其他区域将介入并服务故障区域中的所有用户。  
分布式。分布式意味着数据库不是在一台大计算机上运行，​​而是在多台计算机上运行。用户的数据被复制到多台计算机，因此如果一台甚至两台保存用户数据的计算机发生故障，用户的数据将是安全的。事实上，用户的数据会复制到所有三个区域。这样，如果一个区域出现故障，当新区域准备好开始使用它时，用户的数据就会在那里。  
可扩展。可扩展意味着数据库可以处理系统想要放入的尽可能多的数据。这是分布式数据库的主要优势之一。可以根据需要添加更多计算机以处理更多数据。  
  
Netflix 视频分发  
分发意味着视频文件通过网络从中央位置复制并存储在世界各地的计算机上。对于 Netflix，存储视频的中心位置是 S3。CDN 背后的想法很简单：通过在全球范围内传播计算机，让视频尽可能靠近用户。当用户想要观看视频时，找到最近的带有视频的计算机并从那里流式传输到设备。每个有计算机存储视频内容的位置称为 PoP 或入网点。每个 PoP 都是提供互联网访问的物理位置。它包含服务器、路由器和其他电信设备。  
Netflix 开发了自己的视频存储计算机系统。Netflix 称它们为 Open Connect 设备或 OCA。每个 OCA 都是一个快速的服务器，经过高度优化，可用于传输大文件，并带有大量用于存储视频的硬盘或闪存驱动器。从硬件的角度来看，OCA 没有什么特别之处。它们基于商用 PC 组件，并由各种供应商组装在定制机箱中。从软件的角度来看，OCA 使用 FreeBSD 操作系统和 NGINX 作为 Web 服务器。是的，每个 OCA 都有一个 Web 服务器。通过 NGINX 提供视频流。其他视频服务，如 YouTube 和亚马逊，在他们自己的骨干网络上提供视频。这些公司实际上建立了自己的全球网络，用于向用户提供视频。这样做非常复杂且非常昂贵。Netflix 采用了完全不同的方法来构建其 CDN。Netflix 不运营自己的网络；它也不再运营自己的数据中心。相反，互联网服务提供商 (ISP) 同意将 OCA 放入其数据中心。OCA 免费提供给 ISP 以嵌入到他们的网络中。Netflix 还将 OCA 放置在互联网交换位置 (IXP) 中或附近。ISP 是用户的互联网提供商，它可能是 Verizon、Comcast、AT&T 或数千种其他服务。OCA 放置在 ISP 数据中心里可使得 Netflix 和 ISP 共赢（降低 ISP 的网络资源成本）。   

