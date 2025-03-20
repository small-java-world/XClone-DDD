# Xクローンアプリ開発：DDDとクリーンアーキテクチャによる実装

## はじめに

本記事では、当社で取り組んだXクローンアプリ開発プロジェクトについて技術的側面から解説します。このプロジェクトでは、拡張性と保守性を重視し、ドメイン駆動設計（DDD）とクリーンアーキテクチャを採用しました。また、最新のテクノロジースタックと開発プラクティスを取り入れることで、高品質なアプリケーションの構築を目指しました。

## アーキテクチャ概要

### ドメイン駆動設計（DDD）の適用

DDDの核心は「ドメインモデル」にあります。私たちはビジネスの本質的な課題に焦点を当て、以下のドメイン境界を特定しました：

- ユーザー管理（user）
- 投稿管理（post）
- タイムライン（timeline）
- 通知（notification）
- フォロー関係（relationship）

各ドメインは独立したモジュールとして実装し、ドメイン間の依存関係を最小限に抑えています。

### クリーンアーキテクチャの実装

クリーンアーキテクチャは、システムを「関心の分離」に基づいて階層化します。内側のレイヤーは外側のレイヤーに依存せず、依存の方向は常に内側に向かいます。

#### レイヤー構造

1. **エンティティ（Entity）レイヤー**
   - ビジネスルールとドメインモデルを含む
   - 他のレイヤーへの依存がない

2. **ユースケース（Use Case）レイヤー**
   - アプリケーション固有のビジネスルールを含む
   - エンティティレイヤーのみに依存

3. **インターフェース・アダプター（Interface Adapters）レイヤー**
   - コントローラー、プレゼンター、ゲートウェイなど
   - ユースケースレイヤーに依存

4. **フレームワーク・ドライバー（Frameworks & Drivers）レイヤー**
   - データベース、Webフレームワーク、外部APIなど
   - インターフェース・アダプターレイヤーに依存

#### プロジェクト構造

```
src/
├── main/
│   ├── kotlin/
│   │   └── com/
│   │       └── example/
│   │           └── xclone/
│   │               ├── domain/                  # ドメインレイヤー
│   │               │   ├── user/
│   │               │   │   ├── entity/          # エンティティ
│   │               │   │   ├── repository/      # リポジトリインターフェース
│   │               │   │   └── service/         # ドメインサービス
│   │               │   ├── post/
│   │               │   ├── timeline/
│   │               │   ├── notification/
│   │               │   └── relationship/
│   │               │
│   │               ├── application/             # ユースケースレイヤー
│   │               │   ├── user/
│   │               │   │   ├── command/         # コマンドオブジェクト
│   │               │   │   ├── dto/            # Data Transfer Objects
│   │               │   │   └── usecase/         # ユースケース実装
│   │               │   ├── post/
│   │               │   ├── timeline/
│   │               │   ├── notification/
│   │               │   └── relationship/
│   │               │
│   │               ├── adapter/                 # アダプターレイヤー
│   │               │   ├── controller/          # REST APIコントローラー
│   │               │   ├── presenter/           # レスポンス変換
│   │               │   ├── repository/          # リポジトリ実装
│   │               │   └── gateway/             # 外部サービス接続
│   │               │
│   │               └── infrastructure/          # インフラストラクチャレイヤー
│   │                   ├── config/              # 設定クラス
│   │                   ├── database/            # データベース関連
│   │                   ├── security/            # 認証・認可
│   │                   └── storage/             # ストレージ
│   │
│   └── resources/
│       ├── db/
│       │   └── migration/                       # Flyway マイグレーションファイル
│       ├── application.yml
│       └── application-dev.yml
│
└── test/
    ├── kotlin/                                  # テストコード
    └── resources/                               # テスト用リソース
```

## バックエンド実装

### 技術スタック詳細

- **言語・フレームワーク**: Kotlin + Spring Boot
- **Javaバージョン**: Java 21
- **ビルド自動化ツール**: Gradle
- **テスト**: Kotest + Mockk
- **マイグレーション**: Flyway
- **ORM**: jOOQ
- **データベース**: MySQL（メイン）、Redis（キャッシュ）
- **ストレージ**: MinIO（S3互換）

### Gradle設定

`build.gradle.kts`:

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.2"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
    id("org.flywaydb.flyway") version "9.22.3"
    id("nu.studer.jooq") version "8.2.1"
}

group = "com.example"
version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_21

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // Kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    
    // Database
    implementation("org.springframework.boot:spring-boot-starter-jooq")
    implementation("mysql:mysql-connector-java:8.0.33")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-mysql")
    
    // Storage
    implementation("software.amazon.awssdk:s3:2.20.156")
    
    // JWT
    implementation("io.jsonwebtoken:jjwt-api:0.11.5")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.11.5")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.11.5")
    
    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(module = "mockito-core")
    }
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.kotest.extensions:kotest-extensions-spring:1.1.3")
    testImplementation("io.mockk:mockk:1.13.8")
    
    // Development tools
    developmentOnly("org.springframework.boot:spring-boot-devtools")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "21"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}

// jOOQ code generation configuration
jooq {
    configurations {
        create("main") {
            jooqConfiguration.apply {
                jdbc.apply {
                    driver = "com.mysql.cj.jdbc.Driver"
                    url = "jdbc:mysql://localhost:3306/xclone"
                    user = "root"
                    password = "password"
                }
                generator.apply {
                    name = "org.jooq.codegen.KotlinGenerator"
                    database.apply {
                        name = "org.jooq.meta.mysql.MySQLDatabase"
                        inputSchema = "xclone"
                    }
                    generate.apply {
                        isPojosAsKotlinDataClasses = true
                        isDeprecated = false
                        isRecords = true
                        isImmutablePojos = true
                        isFluentSetters = true
                    }
                    target.apply {
                        packageName = "com.example.xclone.infrastructure.database.jooq"
                        directory = "${project.buildDir}/generated/source/jooq/main"
                    }
                    strategy.name = "org.jooq.codegen.DefaultGeneratorStrategy"
                }
            }
        }
    }
}
```

### 環境変数設定

`.env.local`:

```properties
# Application
APP_ENV=development
SERVER_PORT=8080

# Database
DB_HOST=localhost
DB_PORT=3306
DB_NAME=xclone
DB_USERNAME=root
DB_PASSWORD=password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# MinIO
MINIO_ENDPOINT=http://localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_BUCKET=xclone

# Security
JWT_SECRET=your-secret-key-here-should-be-very-long-and-secure
JWT_EXPIRATION=86400000

# Logging
LOG_LEVEL=INFO
```

アプリケーション起動時に環境変数を読み込むための設定：

```kotlin
@SpringBootApplication
class XCloneApplication

fun main(args: Array<String>) {
    val dotenv = Dotenv.configure()
        .directory("./")
        .filename(".env.local")
        .load()
    
    dotenv.entries().forEach { (key, value) ->
        System.setProperty(key, value)
    }
    
    runApplication<XCloneApplication>(*args)
}
```

### ドメインモデルの実装例

`User.kt`:

```kotlin
package com.example.xclone.domain.user.entity

import java.time.LocalDateTime
import java.util.UUID

data class User(
    val id: UUID,
    val username: String,
    val displayName: String,
    val email: String,
    val passwordHash: String,
    val bio: String?,
    val profileImageUrl: String?,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
) {
    fun updateProfile(displayName: String, bio: String?, profileImageUrl: String?): User {
        return this.copy(
            displayName = displayName,
            bio = bio,
            profileImageUrl = profileImageUrl,
            updatedAt = LocalDateTime.now()
        )
    }
    
    // ドメインロジックを追加
}
```

### リポジトリインターフェース

`UserRepository.kt`:

```kotlin
package com.example.xclone.domain.user.repository

import com.example.xclone.domain.user.entity.User
import java.util.UUID

interface UserRepository {
    fun findById(id: UUID): User?
    fun findByUsername(username: String): User?
    fun findByEmail(email: String): User?
    fun save(user: User): User
    fun delete(id: UUID)
}
```

### ユースケース実装

`CreateUserUseCase.kt`:

```kotlin
package com.example.xclone.application.user.usecase

import com.example.xclone.application.user.command.CreateUserCommand
import com.example.xclone.domain.user.entity.User
import com.example.xclone.domain.user.repository.UserRepository
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.stereotype.Service
import java.time.LocalDateTime
import java.util.UUID

@Service
class CreateUserUseCase(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder
) {
    fun execute(command: CreateUserCommand): User {
        // ユーザー名とメールアドレスの重複チェック
        userRepository.findByUsername(command.username)?.let {
            throw IllegalArgumentException("Username already exists")
        }
        
        userRepository.findByEmail(command.email)?.let {
            throw IllegalArgumentException("Email already exists")
        }
        
        // パスワードのハッシュ化
        val passwordHash = passwordEncoder.encode(command.password)
        
        // ユーザーの作成
        val now = LocalDateTime.now()
        val user = User(
            id = UUID.randomUUID(),
            username = command.username,
            displayName = command.displayName,
            email = command.email,
            passwordHash = passwordHash,
            bio = null,
            profileImageUrl = null,
            createdAt = now,
            updatedAt = now
        )
        
        // ユーザーの保存
        return userRepository.save(user)
    }
}
```

### コントローラー実装

`UserController.kt`:

```kotlin
package com.example.xclone.adapter.controller

import com.example.xclone.application.user.command.CreateUserCommand
import com.example.xclone.application.user.usecase.CreateUserUseCase
import com.example.xclone.application.user.usecase.GetUserUseCase
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.util.UUID

@RestController
@RequestMapping("/api/users")
class UserController(
    private val createUserUseCase: CreateUserUseCase,
    private val getUserUseCase: GetUserUseCase
) {
    @PostMapping
    fun createUser(@RequestBody request: CreateUserRequest): ResponseEntity<UserResponse> {
        val command = CreateUserCommand(
            username = request.username,
            displayName = request.displayName,
            email = request.email,
            password = request.password
        )
        
        val user = createUserUseCase.execute(command)
        
        val response = UserResponse(
            id = user.id,
            username = user.username,
            displayName = user.displayName,
            bio = user.bio,
            profileImageUrl = user.profileImageUrl,
            createdAt = user.createdAt
        )
        
        return ResponseEntity(response, HttpStatus.CREATED)
    }
    
    @GetMapping("/{id}")
    fun getUser(@PathVariable id: UUID): ResponseEntity<UserResponse> {
        val user = getUserUseCase.execute(id) ?: return ResponseEntity(HttpStatus.NOT_FOUND)
        
        val response = UserResponse(
            id = user.id,
            username = user.username,
            displayName = user.displayName,
            bio = user.bio,
            profileImageUrl = user.profileImageUrl,
            createdAt = user.createdAt
        )
        
        return ResponseEntity(response, HttpStatus.OK)
    }
}
```

## フロントエンド実装

### Next.jsプロジェクト構造

```
frontend/
├── src/
│   ├── app/                         # App Router ディレクトリ
│   │   ├── api/                     # API Routes
│   │   ├── (auth)/                  # 認証関連ページ
│   │   │   ├── login/
│   │   │   └── signup/
│   │   ├── [username]/              # ユーザープロフィールページ
│   │   │   ├── page.tsx
│   │   │   └── status/[id]/         # 投稿詳細ページ
│   │   ├── home/                    # ホームタイムライン
│   │   ├── notifications/           # 通知一覧
│   │   ├── explore/                 # 探索ページ
│   │   └── layout.tsx
│   │
│   ├── components/                  # コンポーネント
│   │   ├── common/                  # 共通コンポーネント
│   │   ├── layouts/                 # レイアウトコンポーネント
│   │   ├── post/                    # 投稿関連コンポーネント
│   │   │   ├── PostCard.tsx         # 投稿カード（Presentational）
│   │   │   └── PostCardContainer.tsx # 投稿カードコンテナ（Container）
│   │   ├── user/                    # ユーザー関連コンポーネント
│   │   └── form/                    # フォームコンポーネント
│   │
│   ├── hooks/                       # カスタムフック
│   │   ├── usePost.ts
│   │   ├── useTimeline.ts
│   │   └── useUser.ts
│   │
│   ├── lib/                         # ユーティリティ関数
│   │   ├── api.ts                   # API呼び出し関数
│   │   └── utils.ts                 # 汎用ユーティリティ
│   │
│   ├── types/                       # 型定義
│   │   ├── post.ts
│   │   ├── user.ts
│   │   └── index.ts
│   │
│   └── styles/                      # スタイル
│       └── globals.css
│
├── public/                          # 静的ファイル
│   ├── images/
│   └── favicon.ico
│
├── tests/                           # テスト
│   ├── unit/                        # ユニットテスト
│   └── e2e/                         # E2Eテスト
│       ├── auth.spec.ts             # 認証テスト
│       └── post.spec.ts             # 投稿テスト
│
├── .env.local                       # 環境変数
├── next.config.js                   # Next.js設定
├── package.json                     # パッケージ設定
└── tsconfig.json                    # TypeScript設定
```

### Container/Presentationalパターンの実装例

**Presentationalコンポーネント**:

```tsx
// src/components/post/PostCard.tsx
import React from 'react';
import { format } from 'date-fns';
import { Post } from '@/types/post';
import { User } from '@/types/user';

interface PostCardProps {
  post: Post;
  author: User;
  isLiked: boolean;
  likeCount: number;
  replyCount: number;
  onLike: () => void;
  onReply: () => void;
  onRepost: () => void;
}

export const PostCard: React.FC<PostCardProps> = ({
  post,
  author,
  isLiked,
  likeCount,
  replyCount,
  onLike,
  onReply,
  onRepost
}) => {
  return (
    <div className="border-b border-gray-200 p-4 hover:bg-gray-50">
      <div className="flex space-x-3">
        <img
          src={author.profileImageUrl || '/images/default-avatar.png'}
          alt={author.displayName}
          className="h-10 w-10 rounded-full"
        />
        <div className="flex-1 min-w-0">
          <div className="flex items-center">
            <p className="font-bold text-gray-900">{author.displayName}</p>
            <p className="ml-2 text-sm text-gray-500">@{author.username}</p>
            <span className="mx-1 text-gray-500">·</span>
            <p className="text-sm text-gray-500">
              {format(new Date(post.createdAt), 'MMM d')}
            </p>
          </div>
          <p className="mt-1 text-gray-900">{post.content}</p>
          {post.mediaUrl && (
            <img
              src={post.mediaUrl}
              alt="Post media"
              className="mt-2 rounded-2xl max-h-80 w-full object-cover"
            />
          )}
          <div className="flex justify-between mt-3 max-w-md">
            <button
              onClick={onReply}
              className="flex items-center text-gray-500 hover:text-blue-500"
            >
              <svg /* SVG for reply icon */></svg>
              <span className="ml-2">{replyCount}</span>
            </button>
            <button
              onClick={onRepost}
              className="flex items-center text-gray-500 hover:text-green-500"
            >
              <svg /* SVG for repost icon */></svg>
            </button>
            <button
              onClick={onLike}
              className={`flex items-center ${isLiked ? 'text-red-500' : 'text-gray-500 hover:text-red-500'}`}
            >
              <svg /* SVG for like icon */></svg>
              <span className="ml-2">{likeCount}</span>
            </button>
            <button className="flex items-center text-gray-500 hover:text-blue-500">
              <svg /* SVG for share icon */></svg>
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};
```

**Containerコンポーネント**:

```tsx
// src/components/post/PostCardContainer.tsx
import React from 'react';
import { useOptimistic } from 'react';
import { PostCard } from './PostCard';
import { usePost } from '@/hooks/usePost';
import { Post } from '@/types/post';

interface PostCardContainerProps {
  postId: string;
}

export const PostCardContainer: React.FC<PostCardContainerProps> = ({ postId }) => {
  const { post, author, isLiked, likeCount, replyCount, likePost, unlikePost } = usePost(postId);
  
  const [optimisticLiked, setOptimisticLiked] = useOptimistic(
    isLiked,
    (state, newValue: boolean) => newValue
  );
  
  const [optimisticLikeCount, setOptimisticLikeCount] = useOptimistic(
    likeCount,
    (state, newValue: number) => newValue
  );
  
  if (!post || !author) {
    return <div className="p-4">Loading...</div>;
  }
  
  const handleLike = async () => {
    if (optimisticLiked) {
      setOptimisticLiked(false);
      setOptimisticLikeCount(optimisticLikeCount - 1);
      await unlikePost();
    } else {
      setOptimisticLiked(true);
      setOptimisticLikeCount(optimisticLikeCount + 1);
      await likePost();
    }
  };
  
  const handleReply = () => {
    // リプライモーダルを開くなどの処理
  };
  
  const handleRepost = () => {
    // リポスト処理
  };
  
  return (
    <PostCard
      post={post}
      author={author}
      isLiked={optimisticLiked}
      likeCount={optimisticLikeCount}
      replyCount={replyCount}
      onLike={handleLike}
      onReply={handleReply}
      onRepost={handleRepost}
    />
  );
};
```

### カスタムフックの実装例

```typescript
// src/hooks/usePost.ts
import { useState, useEffect } from 'react';
import { fetchPost, fetchPostLikes, likePost as apiLikePost, unlikePost as apiUnlikePost } from '@/lib/api';
import { Post } from '@/types/post';
import { User } from '@/types/user';

export function usePost(postId: string) {
  const [post, setPost] = useState<Post | null>(null);
  const [author, setAuthor] = useState<User | null>(null);
  const [isLiked, setIsLiked] = useState(false);
  const [likeCount, setLikeCount] = useState(0);
  const [replyCount, setReplyCount] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const loadPost = async () => {
      try {
        setLoading(true);
        const { post, author, statistics } = await fetchPost(postId);
        const { isLiked } = await fetchPostLikes(postId);
        
        setPost(post);
        setAuthor(author);
        setIsLiked(isLiked);
        setLikeCount(statistics.likeCount);
        setReplyCount(statistics.replyCount);
      } catch (err) {
        setError(err instanceof Error ? err : new Error('Failed to load post'));
      } finally {
        setLoading(false);
      }
    };

    loadPost();
  }, [postId]);

  const likePost = async () => {
    if (!post) return;
    
    try {
      await apiLikePost(post.id);
      setIsLiked(true);
      setLikeCount(prev => prev + 1);
    } catch (err) {
      console.error('Failed to like post:', err);
      // 楽観的UIを元に戻す処理
    }
  };

  const unlikePost = async () => {
    if (!post) return;
    
    try {
      await apiUnlikePost(post.id);
      setIsLiked(false);
      setLikeCount(prev => prev - 1);
    } catch (err) {
      console.error('Failed to unlike post:', err);
      // 楽観的UIを元に戻す処理
    }
  };

  return {
    post,
    author,
    isLiked,
    likeCount,
    replyCount,
    loading,
    error,
    likePost,
    unlikePost
  };
}
```

## テスト戦略

### バックエンドテスト

**ユニットテスト例（Kotest + Mockk）**:

```kotlin
package com.example.xclone.application.user.usecase

import com.example.xclone.application.user.command.CreateUserCommand
import com.example.xclone.domain.user.entity.User
import com.example.xclone.domain.user.repository.UserRepository
import io.kotest.assertions.throwables.shouldThrow
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.shouldBe
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import io.mockk.verify
import org.springframework.security.crypto.password.PasswordEncoder
import java.time.LocalDateTime
import java.util.UUID

class CreateUserUseCaseTest : StringSpec({
    val userRepository = mockk<UserRepository>()
    val passwordEncoder = mockk<PasswordEncoder>()
    val useCase = CreateUserUseCase(userRepository, passwordEncoder)

    "should create a new user when valid data is provided" {
        // Arrange
        val command = CreateUserCommand(
            username = "testuser",
            displayName = "Test User",
            email = "test@example.com",
            password = "password123"
        )
        
        val encodedPassword = "encoded_password"
        every { passwordEncoder.encode(command.password) } returns encodedPassword
        every { userRepository.findByUsername(command.username) } returns null
        every { userRepository.findByEmail(command.email) } returns null
        
        val userSlot = slot<User>()
        every { userRepository.save(capture(userSlot)) } answers { userSlot.captured }
        
        // Act
        val result = useCase.execute(command)
        
        // Assert
        result.username shouldBe command.username
        result.displayName shouldBe command.displayName
        result.email shouldBe command.email
        result.passwordHash shouldBe encodedPassword
        
        verify(exactly = 1) {
            userRepository.findByUsername(command.username)
            userRepository.findByEmail(command.email)
            passwordEncoder.encode(command.password)
            userRepository.save(any())
        }
    }
    
    "should throw exception when username already exists" {
        // Arrange
        val command = CreateUserCommand(
            username = "existinguser",
            displayName = "Existing User",
            email = "test@example.com",
            password = "password123"
        )
        
        val existingUser = User(
            id = UUID.randomUUID(),
            username = command.username,
            displayName = "Another User",
            email = "another@example.com",
            passwordHash = "hash",
            bio = null,
            profileImageUrl = null,
            createdAt = LocalDateTime.now(),
            updatedAt = LocalDateTime.now()
        )
        
        every { userRepository.findByUsername(command.username) } returns existingUser
        
        // Act & Assert
        val exception = shouldThrow<IllegalArgumentException> {
            useCase.execute(command)
        }
        
        exception.message shouldBe "Username already exists"
        
        verify(exactly = 1) {
            userRepository.findByUsername(command.username)
        }
        
        verify(exactly = 0) {
            userRepository.findByEmail(any())
            passwordEncoder.encode(any())
            userRepository.save(any())
        }
    }
})
```

### フロントエンドテスト

**Vitestによるユニットテスト例**:

```typescript
// tests/unit/components/post/PostCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { PostCard } from '@/components/post/PostCard';
import { vi } from 'vitest';

describe('PostCard', () => {
  const mockPost = {
    id: '1',
    content: 'Test post content',
    createdAt: new Date().toISOString(),
    mediaUrl: null
  };
  
  const mockAuthor = {
    id: '1',
    username: 'testuser',
    displayName: 'Test User',
    profileImageUrl: '/images/default-avatar.png'
  };
  
  const mockHandlers = {
    onLike: vi.fn(),
    onReply: vi.fn(),
    onRepost: vi.fn()
  };
  
  beforeEach(() => {
    vi.clearAllMocks();
  });
  
  test('renders post content correctly', () => {
    render(
      <PostCard
        post={mockPost}
        author={mockAuthor}
        isLiked={false}
        likeCount={5}
        replyCount={2}
        onLike={mockHandlers.onLike}
        onReply={mockHandlers.onReply}
        onRepost={mockHandlers.onRepost}
      />
    );
    
    expect(screen.getByText('Test post content')).toBeInTheDocument();
    expect(screen.getByText('Test User')).toBeInTheDocument();
    expect(screen.getByText('@testuser')).toBeInTheDocument();
  });
  
  test('calls onLike when like button is clicked', () => {
    render(
      <PostCard
        post={mockPost}
        author={mockAuthor}
        isLiked={false}
        likeCount={5}
        replyCount={2}
        onLike={mockHandlers.onLike}
        onReply={mockHandlers.onReply}
        onRepost={mockHandlers.onRepost}
      />
    );
    
    const likeButton = screen.getByText('5').closest('button');
    fireEvent.click(likeButton);
    
    expect(mockHandlers.onLike).toHaveBeenCalledTimes(1);
  });
});
```

### E2Eテスト

**Playwrightによるログインテスト例**:

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('should allow user to login', async ({ page }) => {
    // 1. トップページに移動
    await page.goto('/');
    
    // 2. ログインページへのリンクをクリック
    await page.click('text=Log in');
    
    // 3. ログインフォームが表示されることを確認
    await expect(page.locator('h1')).toContainText('Log in to X Clone');
    
    // 4. ログイン情報を入力
    await page.fill('input[name="username"]', 'testuser');
    await page.fill('input[name="password"]', 'password123');
    
    // 5. ログインボタンをクリック
    await page.click('button[type="submit"]');
    
    // 6. ログイン後、ホームページにリダイレクトされることを確認
    await expect(page).toHaveURL('/home');
    
    // 7. ナビゲーションにユーザー名が表示されることを確認
    await expect(page.locator('[data-testid="user-nav"]')).toContainText('Test User');
  });
});
```

**投稿作成のE2Eテスト例**:

```typescript
// tests/e2e/post.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Post Creation', () => {
  test.beforeEach(async ({ page }) => {
    // 各テスト前にログイン処理を実行
    await page.goto('/login');
    await page.fill('input[name="username"]', 'testuser');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL('/home');
  });
  
  test('should allow user to create a new post', async ({ page }) => {
    // 1. ホーム画面で新規投稿フォームを確認
    const postForm = page.locator('[data-testid="post-form"]');
    await expect(postForm).toBeVisible();
    
    // 2. 投稿内容を入力
    const postContent = 'This is a test post created during E2E testing! #playwright';
    await postForm.locator('textarea').fill(postContent);
    
    // 3. 投稿ボタンをクリック
    await postForm.locator('button[type="submit"]').click();
    
    // 4. 投稿が成功し、タイムラインに表示されることを確認
    await expect(page.locator('[data-testid="post-item"]').first()).toContainText(postContent);
    
    // 5. カウンターがリセットされることを確認
    await expect(postForm.locator('textarea')).toBeEmpty();
  });
});
```

## 開発環境

### Docker Compose設定

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  # バックエンドアプリケーション
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    container_name: xclone-backend
    ports:
      - "8080:8080"
    volumes:
      - ./backend:/app
      - ~/.gradle:/root/.gradle
    environment:
      - SPRING_PROFILES_ACTIVE=dev
    env_file:
      - .env.local
    depends_on:
      - mysql
      - redis
      - minio
    networks:
      - xclone-network

  # フロントエンドアプリケーション
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    container_name: xclone-frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - NEXT_PUBLIC_API_URL=http://localhost:8080/api
    env_file:
      - .env.local
    depends_on:
      - backend
    networks:
      - xclone-network

  # MySQL データベース
  mysql:
    image: mysql:8.0
    container_name: xclone-mysql
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./docker/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=xclone
    networks:
      - xclone-network

  # Redis キャッシュ
  redis:
    image: redis:7.0-alpine
    container_name: xclone-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - xclone-network

  # MinIO ストレージ
  minio:
    image: minio/minio
    container_name: xclone-minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    command: server /data --console-address ":9001"
    networks:
      - xclone-network

networks:
  xclone-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
  minio-data:
```

## まとめ

本記事では、Xクローンアプリケーションの開発に採用したDDDとクリーンアーキテクチャの実装について解説しました。これらの設計アプローチによって、システムの各レイヤーが明確に分離され、テスト容易性と保守性の高いコードベースが実現しました。

特に以下の点に注目しています：

1. **ドメイン境界の明確化**: 機能ごとにドメインを分割し、依存関係を最小限に抑えることで変更の影響範囲を制限
2. **クリーンアーキテクチャの徹底**: 依存関係が内側に向かうよう設計することで、フレームワークやインフラの変更に強いシステムを構築
3. **Container/Presentationalパターン**: ビジネスロジックとUIの分離による再利用性の向上
4. **React19の新機能活用**: useOptimisticなどの新Hooksを活用した最新のUI実装
5. **テスト駆動開発**: 自動テストによる品質担保と継続的なリファクタリングの実現

今後もこのプロジェクトを通じて得られた知見を共有していく予定です。