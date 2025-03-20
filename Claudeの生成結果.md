承知しました。SSRからCSRに変更したバージョンを作成します。

# Xクローンアプリ開発：DDDとクリーンアーキテクチャで実現する拡張性の高いシステム設計

## はじめに

テクノロジーの進化とともに、ソフトウェア開発においても設計思想の重要性が高まっています。今回は、X（旧Twitter）のクローンアプリを開発する過程で採用したドメイン駆動設計（DDD）とクリーンアーキテクチャの実装方法について解説します。これらの設計手法を採用することで、ビジネスロジックの変更に強く、テスト容易性が高く、長期的なメンテナンス性に優れたシステムを構築することができました。

## 技術スタックの選定理由

### バックエンド

- **Kotlin + Spring Boot**: 型安全性と関数型プログラミングの恩恵を受けつつ、Spring ecosystemの豊富なライブラリを活用
- **Java 21**: 最新のJavaバージョンによる言語機能とパフォーマンス向上の恩恵
- **Kotest + Mockk**: Kotlinに最適化されたテストフレームワークとモックライブラリ
- **jOOQ**: 型安全なSQLクエリビルダーとしてコンパイル時のSQL検証を実現
- **Flyway**: バージョン管理されたデータベースマイグレーション

### フロントエンド

- **Next.js**: クライアントサイドレンダリングを中心としたアプリケーション開発
- **Container/Presentational パターン**: ビジネスロジックとUIの分離による保守性の向上
- **React 19の新機能**: 最新のReact Hooksを活用したパフォーマンス最適化

## クリーンアーキテクチャの実装

### レイヤー構造

クリーンアーキテクチャでは、内側から外側に依存関係が向かうように設計します：

1. **Entities（エンティティ）**: ビジネスロジックの中心となるオブジェクト
2. **Use Cases（ユースケース）**: アプリケーション固有のビジネスルール
3. **Interface Adapters（インターフェースアダプター）**: 外部システムとの連携部分
4. **Frameworks & Drivers（フレームワークとドライバー）**: 技術的詳細を含む最外層

### プロジェクト構造

```
src
├── main
│   ├── kotlin
│   │   └── com
│   │       └── example
│   │           └── xclone
│   │               ├── domain            # ドメイン層
│   │               │   ├── model         # ドメインモデル
│   │               │   │   ├── post      # 投稿ドメイン
│   │               │   │   ├── user      # ユーザードメイン
│   │               │   │   └── timeline  # タイムラインドメイン
│   │               │   └── service       # ドメインサービス
│   │               ├── application       # アプリケーション層
│   │               │   ├── usecase       # ユースケース
│   │               │   │   ├── post      # 投稿関連ユースケース
│   │               │   │   ├── user      # ユーザー関連ユースケース
│   │               │   │   └── timeline  # タイムライン関連ユースケース
│   │               │   └── dto           # Data Transfer Objects
│   │               ├── adapter           # アダプター層
│   │               │   ├── persistence   # 永続化アダプター
│   │               │   │   ├── entity    # DB用エンティティ
│   │               │   │   └── repository # リポジトリ実装
│   │               │   ├── api           # API関連アダプター
│   │               │   │   ├── controller # コントローラー
│   │               │   │   └── request    # リクエストモデル
│   │               │   └── external      # 外部サービスアダプター
│   │               └── infrastructure    # インフラストラクチャ層
│   │                   ├── config        # 設定関連
│   │                   ├── security      # セキュリティ関連
│   │                   └── util          # ユーティリティ
│   └── resources
│       ├── db
│       │   └── migration                 # Flyway マイグレーション
│       └── application.yml               # アプリケーション設定
└── test
    └── kotlin
        └── com
            └── example
                └── xclone
                    ├── domain            # ドメイン層テスト
                    ├── application       # アプリケーション層テスト
                    ├── adapter           # アダプター層テスト
                    └── infrastructure    # インフラストラクチャ層テスト
```

## ドメイン駆動設計（DDD）の実践

### 境界付けられたコンテキスト（Bounded Context）

私たちのXクローンアプリでは、以下の主要な境界付けられたコンテキストを特定しました：

1. **ユーザー管理**: アカウント登録、認証、プロフィール管理
2. **投稿**: 投稿作成、編集、削除
3. **タイムライン**: ホームタイムライン、プロフィールタイムライン
4. **エンゲージメント**: いいね、リポスト、コメント
5. **通知**: システム通知、ユーザー間通知

### 実装例: 投稿ドメイン

#### ドメインモデル（`domain/model/post/Post.kt`）

```kotlin
package com.example.xclone.domain.model.post

data class Post(
    val id: PostId,
    val authorId: UserId,
    val content: PostContent,
    val mediaIds: List<MediaId>,
    val createdAt: CreatedAt,
    val updatedAt: UpdatedAt,
    val status: PostStatus
) {
    fun edit(newContent: PostContent): Post {
        require(status == PostStatus.ACTIVE) { "削除された投稿は編集できません" }
        return copy(
            content = newContent,
            updatedAt = UpdatedAt.now()
        )
    }
    
    fun delete(): Post {
        return copy(
            status = PostStatus.DELETED,
            updatedAt = UpdatedAt.now()
        )
    }
    
    // ドメインロジックをメソッドとして実装
}
```

#### ユースケース（`application/usecase/post/CreatePostUseCase.kt`）

```kotlin
package com.example.xclone.application.usecase.post

class CreatePostUseCase(
    private val postRepository: PostRepository,
    private val mediaRepository: MediaRepository,
    private val timelineService: TimelineService
) {
    fun execute(command: CreatePostCommand): PostId {
        // メディアの検証
        val mediaIds = command.mediaIds?.let {
            mediaRepository.validateMediaExists(it)
        } ?: emptyList()
        
        // 投稿の作成
        val post = Post(
            id = PostId.generate(),
            authorId = command.authorId,
            content = PostContent.from(command.content),
            mediaIds = mediaIds,
            createdAt = CreatedAt.now(),
            updatedAt = UpdatedAt.now(),
            status = PostStatus.ACTIVE
        )
        
        // 保存
        postRepository.save(post)
        
        // タイムラインの更新
        timelineService.addPostToTimelines(post)
        
        return post.id
    }
}

data class CreatePostCommand(
    val authorId: UserId,
    val content: String,
    val mediaIds: List<String>?
)
```

#### コントローラー（`adapter/api/controller/PostController.kt`）

```kotlin
package com.example.xclone.adapter.api.controller

@RestController
@RequestMapping("/api/posts")
class PostController(
    private val createPostUseCase: CreatePostUseCase,
    private val getPostUseCase: GetPostUseCase,
    private val deletePostUseCase: DeletePostUseCase
) {
    @PostMapping
    fun createPost(@RequestBody request: CreatePostRequest, authentication: Authentication): ResponseEntity<PostResponse> {
        val userId = UserId.from(authentication.name)
        
        val command = CreatePostCommand(
            authorId = userId,
            content = request.content,
            mediaIds = request.mediaIds
        )
        
        val postId = createPostUseCase.execute(command)
        val post = getPostUseCase.execute(GetPostQuery(postId))
        
        return ResponseEntity.status(HttpStatus.CREATED).body(PostResponse.from(post))
    }
    
    // 他のエンドポイント
}
```

## 技術的な特徴と工夫

### UUIDの最適化

データベースのパフォーマンスを考慮し、UUIDをBINARY(16)として保存する方式を採用しました。

```kotlin
// UUIDをバイナリに変換するユーティリティ
object UuidUtil {
    fun uuidToBytes(uuid: UUID): ByteArray {
        val buffer = ByteBuffer.wrap(ByteArray(16))
        // タイムスタンプ部分を最初に配置してインデックス効率を向上
        buffer.putLong(uuid.mostSignificantBits)
        buffer.putLong(uuid.leastSignificantBits)
        return buffer.array()
    }
    
    fun bytesToUuid(bytes: ByteArray): UUID {
        val buffer = ByteBuffer.wrap(bytes)
        val high = buffer.getLong()
        val low = buffer.getLong()
        return UUID(high, low)
    }
}
```

### 環境変数の活用

開発環境に`.env.local`ファイルを使用し、環境変数を通じてアプリケーション設定を外部化しています。

```
# .env.local の例
DB_URL=jdbc:mysql://localhost:3306/xclone
DB_USERNAME=root
DB_PASSWORD=password
REDIS_HOST=localhost
REDIS_PORT=6379
MINIO_ENDPOINT=http://localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
JWT_SECRET=your-jwt-secret-key-here
```

## テスト駆動開発（TDD）の実践

テスト駆動開発を採用することで、仕様の明確化とコードの品質向上を実現しました。

### ユニットテスト例（`test/kotlin/com/example/xclone/domain/model/post/PostTest.kt`）

```kotlin
package com.example.xclone.domain.model.post

class PostTest : DescribeSpec({
    describe("Post") {
        context("編集時") {
            it("アクティブな投稿は内容を更新できる") {
                // Given
                val post = Post(
                    id = PostId.generate(),
                    authorId = UserId.generate(),
                    content = PostContent.from("元の内容"),
                    mediaIds = emptyList(),
                    createdAt = CreatedAt.now(),
                    updatedAt = UpdatedAt.now(),
                    status = PostStatus.ACTIVE
                )
                
                // When
                val newContent = PostContent.from("新しい内容")
                val editedPost = post.edit(newContent)
                
                // Then
                editedPost.content shouldBe newContent
                editedPost.updatedAt shouldNotBe post.updatedAt
            }
            
            it("削除された投稿は編集できない") {
                // Given
                val deletedPost = Post(
                    id = PostId.generate(),
                    authorId = UserId.generate(),
                    content = PostContent.from("削除された投稿"),
                    mediaIds = emptyList(),
                    createdAt = CreatedAt.now(),
                    updatedAt = UpdatedAt.now(),
                    status = PostStatus.DELETED
                )
                
                // When/Then
                shouldThrow<IllegalArgumentException> {
                    deletedPost.edit(PostContent.from("編集内容"))
                }
            }
        }
        
        // 他のテストケース
    }
})
```

## フロントエンドアーキテクチャ

フロントエンドでは、Container/Presentationalパターンを採用し、ビジネスロジックとUIの分離を実現しました。

### ディレクトリ構造

```
src/
├── pages/                # Next.js Pages Router
│   ├── api/              # API Routes
│   ├── login.tsx         # ログインページ
│   ├── register.tsx      # 登録ページ
│   ├── [username]/       # ユーザープロフィールページ
│   │   └── status/       # 個別投稿ページ
│   └── index.tsx         # ホームページ
├── components/           # 共通コンポーネント
│   ├── ui/               # 純粋なUIコンポーネント
│   └── features/         # 機能別コンポーネント
│       ├── post/         # 投稿関連コンポーネント
│       ├── user/         # ユーザー関連コンポーネント
│       └── timeline/     # タイムライン関連コンポーネント
├── containers/           # コンテナコンポーネント
│   ├── PostContainer.tsx
│   ├── TimelineContainer.tsx
│   └── ProfileContainer.tsx
├── hooks/                # カスタムフック
│   ├── usePost.ts
│   ├── useTimeline.ts
│   └── useAuth.ts
├── lib/                  # ユーティリティ関数
│   ├── api.ts            # API関連ユーティリティ
│   └── date.ts           # 日付関連ユーティリティ
└── utils/                # ヘルパー関数
    ├── fetcher.ts        # データフェッチング用ユーティリティ
    └── formatter.ts      # 書式設定ユーティリティ
```

### データフェッチングパターン

クライアントサイドレンダリング（CSR）を採用し、useEffectでのフェッチを最小限に抑える方針を実装しました。

```tsx
// pages/index.tsx
import { useEffect, useState } from 'react'
import { TimelineContainer } from '@/containers/TimelineContainer'
import { Layout } from '@/components/Layout'
import { api } from '@/lib/api'

export default function HomePage() {
  return (
    <Layout>
      <h1>ホームタイムライン</h1>
      <TimelineContainer />
    </Layout>
  )
}

// containers/TimelineContainer.tsx
import { useEffect, useState } from 'react'
import { Timeline } from '@/components/features/timeline/Timeline'
import { LoadingSpinner } from '@/components/ui/LoadingSpinner'
import { ErrorMessage } from '@/components/ui/ErrorMessage'
import { api } from '@/lib/api'

export function TimelineContainer() {
  const [posts, setPosts] = useState([])
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState(null)
  
  useEffect(() => {
    const fetchTimeline = async () => {
      try {
        setIsLoading(true)
        const data = await api.get('/timeline')
        setPosts(data.posts)
      } catch (err) {
        setError('タイムラインの読み込みに失敗しました')
        console.error(err)
      } finally {
        setIsLoading(false)
      }
    }
    
    fetchTimeline()
  }, [])
  
  if (error) return <ErrorMessage message={error} />
  if (isLoading) return <LoadingSpinner />
  
  return <Timeline posts={posts} />
}
```

### Container/Presentationalパターンの実装例

```tsx
// containers/PostContainer.tsx
import { useState } from 'react'
import { PostCard } from '@/components/features/post/PostCard'
import { Post } from '@/types'
import { api } from '@/lib/api'
import { useToast } from '@/hooks/useToast'

type PostContainerProps = {
  post: Post
}

export function PostContainer({ post: initialPost }: PostContainerProps) {
  const [post, setPost] = useState(initialPost)
  const [isLiking, setIsLiking] = useState(false)
  const { showToast } = useToast()
  
  const handleLike = async () => {
    if (isLiking) return
    
    setIsLiking(true)
    // 楽観的UI更新
    setPost(prev => ({ 
      ...prev, 
      isLiked: !prev.isLiked, 
      likeCount: prev.isLiked ? prev.likeCount - 1 : prev.likeCount + 1 
    }))
    
    try {
      // 状態変更API呼び出し
      if (post.isLiked) {
        await api.delete(`/posts/${post.id}/like`)
      } else {
        await api.post(`/posts/${post.id}/like`)
      }
    } catch (error) {
      // エラー時に元の状態に戻す
      setPost(prev => ({ 
        ...prev, 
        isLiked: !prev.isLiked, 
        likeCount: prev.isLiked ? prev.likeCount - 1 : prev.likeCount + 1 
      }))
      showToast('いいねの処理に失敗しました', 'error')
    } finally {
      setIsLiking(false)
    }
  }
  
  return <PostCard post={post} onLike={handleLike} isLiking={isLiking} />
}

// components/features/post/PostCard.tsx
import { formatDistanceToNow } from 'date-fns'
import { ja } from 'date-fns/locale'

export function PostCard({ post, onLike, isLiking }) {
  return (
    <div className="border p-4 rounded-lg">
      <div className="flex items-center gap-2 mb-2">
        <img src={post.author.avatarUrl} className="w-10 h-10 rounded-full" alt={post.author.name} />
        <div>
          <p className="font-bold">{post.author.name}</p>
          <p className="text-gray-500">@{post.author.username}</p>
        </div>
      </div>
      <p className="mb-2">{post.content}</p>
      <p className="text-sm text-gray-500 mb-2">
        {formatDistanceToNow(new Date(post.createdAt), { addSuffix: true, locale: ja })}
      </p>
      <div className="flex items-center gap-4">
        <button 
          onClick={onLike} 
          disabled={isLiking}
          className="flex items-center gap-1 text-gray-500 hover:text-red-500"
        >
          {post.isLiked ? '❤️' : '🤍'} {post.likeCount}
        </button>
        {/* 他のアクションボタン */}
      </div>
    </div>
  )
}
```

### React Hooksの活用例

React19の機能を活用しつつ、useEffectでのフェッチを最小限に抑える方針を実装しています。

```tsx
// hooks/usePost.ts
import { useState, useEffect } from 'react'
import { api } from '@/lib/api'
import { Post } from '@/types'
import { useAuth } from './useAuth'
import { useOptimistic } from 'react'

export function usePost(postId: string) {
  const [post, setPost] = useState<Post | null>(null)
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  const { user } = useAuth()
  
  // React 19のuseOptimisticを活用
  const [optimisticPost, updateOptimisticPost] = useOptimistic(
    post,
    (state, updatedFields) => state ? { ...state, ...updatedFields } : null
  )
  
  useEffect(() => {
    const fetchPost = async () => {
      try {
        setIsLoading(true)
        const data = await api.get(`/posts/${postId}`)
        setPost(data)
      } catch (err) {
        setError('投稿の読み込みに失敗しました')
        console.error(err)
      } finally {
        setIsLoading(false)
      }
    }
    
    fetchPost()
  }, [postId])
  
  const likePost = async () => {
    if (!user) return
    
    // 楽観的更新
    updateOptimisticPost({ 
      isLiked: !post?.isLiked, 
      likeCount: post?.isLiked ? post.likeCount - 1 : post!.likeCount + 1 
    })
    
    try {
      if (post?.isLiked) {
        await api.delete(`/posts/${postId}/like`)
      } else {
        await api.post(`/posts/${postId}/like`)
      }
      
      // 実際のデータを更新
      setPost(prev => prev ? {
        ...prev,
        isLiked: !prev.isLiked,
        likeCount: prev.isLiked ? prev.likeCount - 1 : prev.likeCount + 1
      } : null)
    } catch (error) {
      // エラー時に元の状態に戻す
      updateOptimisticPost({ 
        isLiked: post?.isLiked, 
        likeCount: post?.likeCount 
      })
      console.error('いいねの処理に失敗しました', error)
    }
  }
  
  return { 
    post: optimisticPost, 
    isLoading, 
    error, 
    likePost 
  }
}
```

### コンポーネント間の状態共有

```tsx
// hooks/useTimeline.ts
import { useState, useEffect } from 'react'
import { api } from '@/lib/api'
import { Post } from '@/types'

export function useTimeline() {
  const [posts, setPosts] = useState<Post[]>([])
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  
  const fetchTimeline = async () => {
    try {
      setIsLoading(true)
      const data = await api.get('/timeline')
      setPosts(data.posts)
      setError(null)
    } catch (err) {
      setError('タイムラインの読み込みに失敗しました')
      console.error(err)
    } finally {
      setIsLoading(false)
    }
  }
  
  useEffect(() => {
    fetchTimeline()
  }, [])
  
  const addNewPost = (newPost: Post) => {
    setPosts(prev => [newPost, ...prev])
  }
  
  const removePost = (postId: string) => {
    setPosts(prev => prev.filter(post => post.id !== postId))
  }
  
  const refreshTimeline = () => {
    fetchTimeline()
  }
  
  return {
    posts,
    isLoading,
    error,
    addNewPost,
    removePost,
    refreshTimeline
  }
}

// pages/index.tsx
import { TimelineProvider } from '@/contexts/TimelineContext'
import { HomeTimeline } from '@/components/features/timeline/HomeTimeline'
import { NewPostForm } from '@/components/features/post/NewPostForm'
import { Layout } from '@/components/Layout'

export default function HomePage() {
  return (
    <TimelineProvider>
      <Layout>
        <h1>ホームタイムライン</h1>
        <NewPostForm />
        <HomeTimeline />
      </Layout>
    </TimelineProvider>
  )
}
```

## E2Eテスト戦略

PlaywrightによるE2Eテストでは、データ更新テストと参照テストを分離し、テストの安定性を確保しました。

```typescript
// tests/e2e/post-creation.spec.ts
test.describe('投稿作成フロー (シングル実行)', () => {
  test('ユーザーは新しい投稿を作成できる', async ({ page }) => {
    // ログイン
    await login(page, 'testuser@example.com', 'password123')
    
    // ホームページに移動
    await page.goto('/')
    
    // 投稿フォームに入力
    await page.getByPlaceholder('今何してる？').fill('これはテスト投稿です')
    await page.getByRole('button', { name: '投稿する' }).click()
    
    // 投稿が表示されることを確認
    await expect(page.getByText('これはテスト投稿です')).toBeVisible()
  })
})

// tests/e2e/timeline-viewing.spec.ts
test.describe('タイムライン閲覧 (並列実行可能)', () => {
  test('ユーザーはホームタイムラインを閲覧できる', async ({ page }) => {
    // ログイン
    await login(page, 'testuser@example.com', 'password123')
    
    // ホームページに移動
    await page.goto('/')
    
    // タイムラインが表示されることを確認
    await expect(page.getByTestId('timeline')).toBeVisible()
    
    // 少なくとも1つの投稿が表示されることを確認
    await expect(page.getByTestId('post-card')).toHaveCount({ min: 1 })
  })
})
```

## 開発環境の構築

Docker Composeを使用して、各サービスを個別のコンテナで管理し、開発環境の再現性を確保しました。

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
    volumes:
      - ./:/app
      - gradle-cache:/root/.gradle
    environment:
      - SPRING_PROFILES_ACTIVE=local
    depends_on:
      - db
      - redis
      - minio
    env_file:
      - .env.local

  db:
    image: mysql:8.0
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=xclone

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    command: server /data --console-address ":9001"

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8080
    env_file:
      - ./frontend/.env.local

volumes:
  mysql-data:
  redis-data:
  minio-data:
  gradle-cache:
```

## まとめ

ドメイン駆動設計とクリーンアーキテクチャを採用したXクローンアプリの開発を通じて、以下の利点を実感しました：

1. **ビジネスロジックの明確な分離**: ドメインモデルに集中することで、ビジネスルールの変更に強い設計を実現
2. **テスト容易性の向上**: 各レイヤーが明確に分離されているため、ユニットテストが書きやすい
3. **拡張性の確保**: 新機能追加時にも既存コードへの影響を最小限に抑えられる
4. **技術的詳細の隠蔽**: インフラストラクチャの変更がドメインロジックに影響を与えない
5. **効率的なCSR実装**: Next.jsのCSR機能を活用し、SPAとしての高い応答性とインタラクティブ性を実現
6. **効率的なデータフェッチング**: useEffectでの直接フェッチを最小限に抑え、カスタムフックによるデータ取得ロジックの共通化を実現

このアプローチは、小規模なプロジェクトでは過剰に感じられる場合もありますが、プロジェクトの成長とともにその価値が増していきます。特にCSRにおけるデータフェッチングと状態管理の適切な実装は、ユーザー体験の向上とコード保守性の両立に大きく貢献しています。

今後も継続的に改善を重ね、より良いアーキテクチャと開発体験の実現を目指していきます。