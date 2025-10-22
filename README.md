# YT Ads Automator

https://img.shields.io/badge/YouTube-API-red?logo=youtube

https://img.shields.io/badge/Google%2520Ads-API-blue?logo=googleads

https://img.shields.io/badge/Java-11%252B-orange?logo=openjdk

一个高性能、全自动化的Java应用程序，用于批量上传视频到YouTube并自动在Google Ads中创建视频广告。专为效果营销专家和需要大规模投放视频广告的团队设计。

🚀 核心功能
批量视频上传：支持每日上传100+广告创意视频至YouTube，采用多线程并发处理提升上传效率

智能状态监控：基于调度器的自动轮询机制，实时监控视频处理状态，直至视频可公开播放

无缝广告创建：在视频处理完成后，自动获取 videoId 并调用Google Ads API创建相应的视频广告组和广告创意

企业级元数据管理：支持通过配置文件批量设置视频标题、描述、标签、分类和隐私状态

容错与重试机制：内置完善的异常处理、指数退避重试策略和失败任务恢复功能

详细运行日志：提供完整的操作审计日志，便于跟踪每个视频的处理流水线

🛠 技术架构
技术栈
语言: Java 11+

构建工具: Maven 3.6+

核心API:

YouTube Data API v3

Google Ads API

HTTP客户端: Google API Client Library for Java

JSON处理: Gson 2.8+

调度框架: Quartz Scheduler

配置管理: YAML + Typesafe Config

日志框架: SLF4J + Logback

📊 API配额消耗分析
每日配额需求计算
操作类型	单个视频消耗	100视频消耗	频率
videos.insert	1600 units	160,000 units	一次性
videos.list (状态检查)	1-10 units	10,000 units	每5分钟轮询
元数据更新	50 units	5,000 units	按需
总计估算	~1650 units	~175,000 units	每日
配额优化策略
智能轮询：视频处理初期降低检查频率，接近完成时增加频率

批量操作：在可能的情况下使用批量API请求

本地缓存：缓存视频元数据和状态，减少重复API调用

连接池：复用HTTP连接，减少建立连接的开销

🗂 项目结构

src/

├── main/

│   ├── java/

│   │   ├── com/ytads/automator/

│   │   │   ├── YouTubeUploader.java      # YouTube上传服务

│   │   │   ├── VideoStatusPoller.java    # 状态轮询服务

│   │   │   ├── AdsCreator.java          # Google Ads创建服务

│   │   │   ├── ConfigManager.java       # 配置管理

│   │   │   ├── RetryHandler.java        # 重试逻辑

│   │   │   └── MainApplication.java     # 主应用入口

│   │   └── resources/

│   │       ├── application.conf         # 应用配置

│   │       ├── client_secrets.json      # API认证配置

│   │       └── logback.xml             # 日志配置

│   └── scripts/

│       └── upload_batch.sh             # 批量上传脚本

└── test/

    └── java/
    
        └── integration/                 # 集成测试
        
⚙️ 核心代码示例

YouTube上传服务

```java```
public class YouTubeUploader {
    private static final Logger logger = LoggerFactory.getLogger(YouTubeUploader.class);
    
    public String uploadVideo(File videoFile, VideoMetadata metadata) {
        try {
            YouTube youtube = getYouTubeService();
            Video video = new Video();
            VideoSnippet snippet = new VideoSnippet();
            VideoStatus status = new VideoStatus();
            
            snippet.setTitle(metadata.getTitle());
            snippet.setDescription(metadata.getDescription());
            snippet.setTags(metadata.getTags());
            video.setSnippet(snippet);
            
            status.setPrivacyStatus("private"); // 广告素材设为私有
            video.setStatus(status);
            
            YouTube.Videos.Insert request = youtube.videos()
                .insert("snippet,status", video, 
                        new FileContent("video/*", videoFile));
            
            Video uploadedVideo = request.execute();
            logger.info("视频上传成功: ID={}, Title={}", 
                uploadedVideo.getId(), uploadedVideo.getSnippet().getTitle());
            
            return uploadedVideo.getId();
            
        } catch (GoogleJsonResponseException e) {
            logger.error("YouTube API错误: {}", e.getDetails().getMessage());
            throw new RuntimeException("上传失败", e);
        } catch (IOException e) {
            logger.error("IO错误 during upload", e);
            throw new RuntimeException("上传失败", e);
        }
    }
}
```
状态轮询服务
```java```
@Component
public class VideoStatusPoller {
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(5);
    
    public CompletableFuture<VideoProcessingStatus> pollUntilReady(String videoId) {
        return CompletableFuture.supplyAsync(() -> {
            while (true) {
                try {
                    Video video = getVideoStatus(videoId);
                    String status = video.getStatus().getUploadStatus();
                    
                    if ("processed".equals(status)) {
                        logger.info("视频处理完成: {}", videoId);
                        return VideoProcessingStatus.READY;
                    } else if ("failed".equals(status)) {
                        logger.error("视频处理失败: {}", videoId);
                        return VideoProcessingStatus.FAILED;
                    }
                    
                    // 指数退避策略
                    Thread.sleep(getNextPollingInterval());
                    
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return VideoProcessingStatus.INTERRUPTED;
                } catch (Exception e) {
                    logger.warn("轮询失败, 重试中: {}", e.getMessage());
                }
            }
        }, scheduler);
    }
}

```
Google Ads创建服务
```java```
@Service
public class AdsCreator {
    
    public void createVideoAd(String videoId, AdCampaignConfig config) {
        try (GoogleAdsClient googleAdsClient = GoogleAdsClient.newBuilder()
            .fromPropertiesFile("google-ads.properties")
            .build()) {
            
            // 创建视频广告素材
            String adResourceName = createVideoAdResource(googleAdsClient, videoId, config);
            logger.info("广告素材创建成功: {}", adResourceName);
            
            // 创建广告组并关联素材
            String adGroupResourceName = createAdGroup(googleAdsClient, config);
            associateAdWithGroup(googleAdsClient, adResourceName, adGroupResourceName);
            
            logger.info("视频广告创建完成: videoId={}, adGroup={}", 
                videoId, adGroupResourceName);
                
        } catch (GoogleAdsException gae) {
            logger.error("Google Ads API请求失败: {}", gae.getGoogleAdsFailure());
            throw new RuntimeException("广告创建失败", gae);
        }
    }
}
```
🔧 安装与配置
前置要求
Java 11 或更高版本

Maven 3.6+

YouTube Data API 凭据

Google Ads API 开发者令牌

有效的OAuth 2.0客户端ID

配置认证
在Google Cloud Console中启用YouTube Data API v3和Google Ads API

下载OAuth 2.0客户端凭据，保存为 src/main/resources/client_secrets.json

配置Google Ads API属性文件 google-ads.properties

📈 使用示例
批量上传执行
```java```
public class BatchUploadExample {
    public static void main(String[] args) {
        YTAdsAutomator automator = new YTAdsAutomator();
        
        // 配置批量任务
        BatchUploadConfig config = BatchUploadConfig.builder()
            .videoDirectory("/path/to/ad/creatives")
            .metadataFile("/path/to/metadata.csv")
            .maxConcurrentUploads(5)
            .pollingInterval(Duration.ofMinutes(5))
            .build();
        
        // 执行批量上传和广告创建
        BatchUploadResult result = automator.executeBatchUpload(config);
        
        logger.info("批量任务完成: 成功={}, 失败={}", 
            result.getSuccessCount(), result.getFailureCount());
    }
}
```
🤝 为配额申请提供的证明
本项目是一个真实运行的生产级应用，当前的API配额限制严重影响了我们的广告运营效率。每日10,000的配额在处理约6个视频后就会耗尽，无法支持我们为客户管理100+视频广告的需求。

通过此README和配套的代码库，我们证明了：

技术可行性：完整的Java实现，包含错误处理和最佳实践

业务必要性：清晰的广告自动化流水线，服务于真实的营销需求

配额合理性：基于实际业务量的详细配额计算

合规承诺：严格遵守YouTube和Google Ads API的服务条款

我们恳请审核团队批准我们的配额提升申请，以支持我们在Google广告生态中的持续投入和增长。

📄 许可证
MIT License - 详见 LICENSE 文件。

注意: 在实际使用前，请确保您已获得所有必要的API权限并遵守YouTube API服务条款和Google Ads API政策。
