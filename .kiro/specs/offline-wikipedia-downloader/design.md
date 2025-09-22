# 设计文档

## 概述

离线维基百科下载器功能将为 WiKi 应用添加完整的离线维基百科管理系统。该功能包括从 Kiwix 镜像站点获取可用文件列表、下载管理、本地存储和离线内容查看。设计采用 MVVM 架构模式，确保代码的可维护性和可测试性。

## 架构

### 整体架构
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Presentation  │    │    Business     │    │      Data       │
│     Layer       │◄──►│     Logic       │◄──►│     Layer       │
│   (SwiftUI)     │    │   (ViewModels)  │    │ (Repositories)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 核心组件
- **DownloadListView**: 主要的下载管理界面
- **WikipediaFileService**: 处理文件列表获取和下载逻辑
- **ZimFileManager**: 管理本地 .zim 文件的存储和访问
- **WikipediaReader**: 处理 .zim 文件的读取和内容展示

## 组件和接口

### 1. 数据模型

#### WikipediaFile
```swift
struct WikipediaFile: Identifiable, Codable {
    let id = UUID()
    let name: String
    let url: URL
    let size: Int64
    let language: String
    let description: String
    let publishDate: Date
    var downloadStatus: DownloadStatus = .notDownloaded
    var downloadProgress: Double = 0.0
}

enum DownloadStatus: String, CaseIterable {
    case notDownloaded = "未下载"
    case downloading = "下载中"
    case paused = "已暂停"
    case completed = "已完成"
    case failed = "下载失败"
}
```

#### DownloadTask
```swift
class DownloadTask: ObservableObject {
    let file: WikipediaFile
    @Published var progress: Double = 0.0
    @Published var status: DownloadStatus = .notDownloaded
    @Published var downloadSpeed: String = ""
    @Published var error: Error?
    
    private var urlSessionTask: URLSessionDownloadTask?
}
```

### 2. 服务层

#### WikipediaFileService
```swift
protocol WikipediaFileServiceProtocol {
    func fetchAvailableFiles() async throws -> [WikipediaFile]
    func downloadFile(_ file: WikipediaFile) -> DownloadTask
    func cancelDownload(for file: WikipediaFile)
    func pauseDownload(for file: WikipediaFile)
    func resumeDownload(for file: WikipediaFile)
}

class WikipediaFileService: NSObject, WikipediaFileServiceProtocol, URLSessionDownloadDelegate {
    private let session: URLSession
    private var activeTasks: [UUID: DownloadTask] = [:]
    
    // 实现网络请求和下载管理
}
```

#### ZimFileManager
```swift
protocol ZimFileManagerProtocol {
    func getLocalFiles() -> [WikipediaFile]
    func deleteFile(_ file: WikipediaFile) throws
    func getFileSize(_ file: WikipediaFile) -> Int64
    func isFileValid(_ file: WikipediaFile) -> Bool
    func getStorageInfo() -> StorageInfo
}

struct StorageInfo {
    let totalSpace: Int64
    let availableSpace: Int64
    let usedSpace: Int64
}
```

### 3. 视图模型

#### DownloadListViewModel
```swift
class DownloadListViewModel: ObservableObject {
    @Published var availableFiles: [WikipediaFile] = []
    @Published var downloadedFiles: [WikipediaFile] = []
    @Published var isLoading = false
    @Published var searchText = ""
    @Published var selectedLanguage: String?
    @Published var sortOption: SortOption = .name
    @Published var errorMessage: String?
    
    private let fileService: WikipediaFileServiceProtocol
    private let zimManager: ZimFileManagerProtocol
    
    // 业务逻辑方法
    func loadAvailableFiles()
    func downloadFile(_ file: WikipediaFile)
    func deleteFile(_ file: WikipediaFile)
    func filteredFiles() -> [WikipediaFile]
}

enum SortOption: String, CaseIterable {
    case name = "名称"
    case size = "大小"
    case date = "日期"
    case language = "语言"
}
```

### 4. 用户界面组件

#### DownloadListView
```swift
struct DownloadListView: View {
    @StateObject private var viewModel = DownloadListViewModel()
    @State private var showingDownloaded = false
    
    var body: some View {
        NavigationView {
            VStack {
                SearchAndFilterBar()
                SegmentedControl() // 可用文件 / 已下载文件
                FileListView()
            }
        }
    }
}
```

#### FileRowView
```swift
struct FileRowView: View {
    let file: WikipediaFile
    let onDownload: () -> Void
    let onDelete: () -> Void
    
    var body: some View {
        HStack {
            FileInfoView(file: file)
            Spacer()
            ActionButtonsView(file: file, onDownload: onDownload, onDelete: onDelete)
        }
    }
}
```

## 数据模型

### 本地存储结构
```
Documents/
├── WikipediaFiles/
│   ├── downloaded/
│   │   ├── wikipedia_zh_all_2024-01.zim
│   │   └── wikipedia_en_all_2024-01.zim
│   └── metadata/
│       └── files_metadata.json
```

### 网络数据解析
从 Kiwix 镜像站点解析 HTML 页面，提取 .zim 文件信息：
- 文件名和下载链接
- 文件大小
- 最后修改日期
- 通过文件名推断语言和描述

## 错误处理

### 网络错误处理
```swift
enum NetworkError: LocalizedError {
    case noInternetConnection
    case serverUnavailable
    case invalidResponse
    case downloadFailed(Error)
    
    var errorDescription: String? {
        switch self {
        case .noInternetConnection:
            return "网络连接不可用，请检查网络设置"
        case .serverUnavailable:
            return "服务器暂时不可用，请稍后重试"
        case .invalidResponse:
            return "服务器响应无效"
        case .downloadFailed(let error):
            return "下载失败：\(error.localizedDescription)"
        }
    }
}
```

### 存储错误处理
```swift
enum StorageError: LocalizedError {
    case insufficientSpace
    case fileCorrupted
    case permissionDenied
    case fileNotFound
    
    var errorDescription: String? {
        switch self {
        case .insufficientSpace:
            return "存储空间不足，请删除一些文件后重试"
        case .fileCorrupted:
            return "文件已损坏，建议重新下载"
        case .permissionDenied:
            return "没有文件访问权限"
        case .fileNotFound:
            return "文件未找到"
        }
    }
}
```

## 测试策略

### 单元测试
- **WikipediaFileService**: 测试网络请求、下载逻辑
- **ZimFileManager**: 测试文件管理操作
- **DownloadListViewModel**: 测试业务逻辑和状态管理

### 集成测试
- **网络层集成**: 测试实际的网络请求和响应处理
- **文件系统集成**: 测试文件下载、存储和删除流程
- **UI集成**: 测试用户交互和状态更新

### UI测试
- **下载流程**: 测试完整的文件下载用户流程
- **搜索和筛选**: 测试搜索功能和筛选器
- **错误处理**: 测试各种错误场景的用户体验

### 性能测试
- **大文件下载**: 测试大型 .zim 文件的下载性能
- **内存使用**: 监控下载过程中的内存使用情况
- **并发下载**: 测试多个文件同时下载的性能

### 测试数据
- 使用模拟的 Kiwix 服务器响应进行单元测试
- 创建小型测试 .zim 文件用于集成测试
- 使用真实的网络环境进行端到端测试