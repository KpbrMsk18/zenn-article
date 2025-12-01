---
title: "フロントエンドエンジニアがAIの力を借りてバックエンドの実装を学んだ話"
emoji: "🏃‍♂️"
type: "tech"
topics: ["nestjs", "typescript", "backend", "nodejs", "初心者向け"]
published: false
publication_name: "uguisu_blog"
---

[うぐいすソリューションズ Advent Calendar](https://qiita.com/advent-calendar/2025/uguisu-solutions) 11 日目。担当のかぴばらです！

私はこれまでフロントエンドを主に担当してきたというのもあり、バックエンドの知識がほぼ皆無でした。
アプリ開発をしたいなと思ってはいたのですが、バックエンドの正しい実装がわからないからバックエンドのレビューをしようがないよなぁなんて考えてはや幾年。
作りはしたものの、バックエンドはほぼ眺めるだけになっていました。

そんな中、社内でとある記事をベースにバックエンドの型安全の話をしている時に「サービスロジックで型チェックしているのが違和感を感じる」という話が出ました。型ってどこでもチェックするもんだと思っていたので、少し衝撃を受けて、改めて聞くと各コンポーネントに責務があるという話が出てきました。その話はなんとなくは知っていたのですが型のチェックもある程度責務で分けられるという話は新鮮でした。

その話の中で「バックエンドあんまり触ったことないんですよねぇ」と相談していたら、作って AI に解説して貰えばいいというアドバイスをいただき、そうしてみるかと思い立ってやってみたという流れになります。

ほぼ知らない分野の学習に AI を使ったその過程をお見せすることで、誰かのお役にたてればというのと、自分自身の理解促進のために記事化することにしました！

前置きが長くなりましたが目次は以下になります。

## 🎯 目次

1. [技術選定について](#1-技術選定について)
2. [NestJS アーキテクチャの全体像](#2-nestjsアーキテクチャの全体像)
3. [リクエスト処理の流れ](#3-リクエスト処理の流れ)
4. [主要コンポーネント](#4-主要コンポーネント)
5. [依存性注入(DI)](#5-依存性注入di)
6. [データベース操作](#6-データベース操作)
7. [まとめ](#7-まとめ)

---

## 1. 技術選定について

どうして、バックエンドのスタックで NestJS を選択したかというと以下が当てはまって自分には理解しやすいかなと思ったからです。

- 私自身が Typescript をよく使うこと
- NestJS を使ったことがあったこと

Java も遠い昔に経験があるのですが、Java8 の知識で止まっているというのもあり、 NestJS の方が記憶が新しいので理解しやすそうだなという判断です。

## 2. NestJS アーキテクチャの全体像

### レイヤードアーキテクチャ

バックエンドを知る上で、アーキテクチャの話はなんとなく理解できていたのですが再度復習。
NestJS では以下のようなレイヤードアーキテクチャをとることが一般的なようです。
フローで示すと以下のような流れになっており、データの流れがわかりやすいですね。
フロントエンドをしているとなかなかデータの流れを追いにくい部分があったりするので、構成ではっきりわかる形になっているとすごく美しく感じますね！

```
┌─────────────────────────────────┐
│   Controller                    │
└─────────────────────────────────┘
           ↓
┌─────────────────────────────────┐
│   Service                       │
└─────────────────────────────────┘
           ↓
┌─────────────────────────────────┐
│   Repository                    │
└─────────────────────────────────┘
           ↓
┌─────────────────────────────────┐
│   Database                      │
└─────────────────────────────────┘
```

### 主な責任分担

| レイヤー       | 責任             | 例                         |
| -------------- | ---------------- | -------------------------- |
| **Controller** | ルーティング     | HTTP リクエストを受け取る  |
| **Service**    | ビジネスロジック | メールアドレス重複チェック |
| **Repository** | データアクセス   | データベースから取得・保存 |

---

## 3. リクエスト処理の流れ

レイヤードアーキテクチャまでは記憶があったのですが、私の場合はリクエストの流れについての理解があまり正確ではありませんでした。

理解していなかったポイントとしては、

- Middleware の実行タイミングと実行時に何をするのか
- Guard って何？
- Pipe って何？

みたいなところが主に理解できていませんでした。

まずは実行順序から...

### 実行順序

```
リクエスト
    ↓
1. Middleware          リクエスト前処理
    ↓
2. Guard               認証・認可
    ↓
3. Interceptor(Before) リクエスト前処理
    ↓
4. Pipe                データ変換・バリデーション
    ↓
5. Controller          ハンドラー実行
    ↓
6. Service             ビジネスロジック
    ↓
7. Repository          データベース操作
    ↓
8. Interceptor(After)  レスポンス後処理
    ↓
レスポンス
```

「コントローラに行くまでにこんなに処理挟むんだ」と少し驚きました。

### 各コンポーネントの役割

各コンポーネントの役割については以下の表の通りです。

確かに、NestJS で実装すると Guard や Pipe などを使ってバリデーションしているということがわかりました。
実際にソース見るとすごくわかりやすかったです。

| コンポーネント  | タイミング             | 用途                          |
| --------------- | ---------------------- | ----------------------------- |
| **Middleware**  | 最初                   | ロギング、CORS、リクエスト ID |
| **Guard**       | Middleware 後          | 認証・認可                    |
| **Interceptor** | Guard 後/Controller 後 | レスポンス変換、キャッシュ    |
| **Pipe**        | Controller 前          | 型変換、バリデーション        |

---

## 3. 主要コンポーネント

私が（主に Claude がですが）作った実際のコードを用いてみていきましょう。

### 3.1 Module (モジュール)

まずはモジュールから。
モジュールとはアプリケーションを構成する基本単位のことを指すようです。
関連する機能を 1 つのまとまりとしてグループ化するための仕組み。
user というモジュールでは UserController を使っていて、サービスは UserService というのを使っているよ！
みたいなのを登録しています。
exports することで、他のモジュールでも UserService を利用可能になっているということです！

要するにこの機能に関連するもの（コントローラとかサービス）はこれだよ！みたいなのを見えやすくしているということですね！
あとは、DRY 原則に基づいて同じ実装している箇所をまとめられるということでした。

あとはカプセル化して他の機能がその機能の詳細を知らなくても IF だけわかればいい状態にできるというメリットもあります。
きゃー！素敵ー！

**役割:** 機能をグループ化

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}
```

**ポイント:**

- `imports`: 他のモジュールをインポート
- `controllers`: コントローラーを登録
- `providers`: サービスを登録
- `exports`: 他のモジュールで使えるようにする

---

### 3.2 Controller (コントローラー)

そしてコントローラです。
リクエストを受け取ってどのサービス（業務ロジック？）を実行するかを振り分ける役割があります。

そしてここで `@UseGuards`というアノテーションを使って利用されるのが Guard というものになります。
他にも DTO に定義された通りの型になっているかを検証する Pipe というものもあります。
NestJS では RequestBody の DTO に定義された通りに型チェックが走るようになっているようでした。便利すぎるだろ。。。

Guard や Pipe についての説明は後ほどします。

**役割:** HTTP リクエストを受け取る

```typescript
@Controller("users")
@UseGuards(JwtAuthGuard)
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  async findAll(): Promise<SuccessResponseDto<UserResponseDto[]>> {
    const users = await this.userService.findAll();
    return new SuccessResponseDto(users, "Users retrieved successfully");
  }

  @Get("profile")
  async getProfile(@Request() req: AuthenticatedRequest) {
    const user = await this.userService.findById(req.user.id);
    return new SuccessResponseDto(user, "Profile retrieved successfully");
  }

  @Patch("password")
  async changePassword(
    @Request() req: AuthenticatedRequest,
    @Body() changePasswordDto: ChangePasswordDto
  ) {
    await this.userService.changePassword(req.user.id, changePasswordDto);
    return new SuccessResponseDto(null, "Password changed successfully");
  }
}
```

**ポイント:**

- `@Controller('users')`: `/users`のルートを処理
- `@Get()`, `@Post()`, `@Patch()`: HTTP メソッド
- `@UseGuards()`: ガードを適用
- `@Body()`, `@Request()`: リクエストデータを取得

---

### 3.3 Service (サービス)

そしてサービスになります。
ここでは主に業務ロジックを書いていくことになり書いていくことになります。
あとは、その検証ですかね。
検証というのは「存在しているユーザかどうかをチェックして存在していなければエラーを出す」みたいなことです。
簡単にいうと、業務的に正しいかをチェックしています。
あとは、ここであまり DB の操作の実装をすることはなく、TypeORM などの機能をつかって DB アクセスしたりカスタマイズで Repository という層を作ったりして DB アクセス部分を外に出したりします。
ここで重要なのは「UserService という機能が DB アクセスの時の詳細を知らないこと」です。
IF だけ知っていて、どの値を渡すかだけ知っていればいいみたいなことですね。

**@Injectable() について**

`@Injectable()`アノテーションは、ざっくり言うと「Nest さん！このクラスを管理してぇ！」ってことです。
このアノテーションを使うことで、DI コンテナに登録され、必要な時に自動的にインスタンス化されて注入されます。
つまり、`new UserService()`のように自分でインスタンスを作る必要がなくなります。

また、`@Injectable()`を付けたクラスは、デフォルトで**シングルトン**（アプリケーション全体で 1 つのインスタンスのみ）として管理されるため、不要にインスタンスが重複するのを防いでくれます。

DI って便利だなぁと感動しました（KONAMI）

**役割:** ビジネスロジック

```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>
  ) {}

  async create(registerDto: RegisterDto): Promise<UserResponseDto> {
    // ビジネスロジック: メールアドレスの重複チェック
    const existingUser = await this.userRepository.findOne({
      where: { email: registerDto.email },
    });

    if (existingUser) {
      throw new ConflictException("Email already registered");
    }

    // データベースに保存
    const user = this.userRepository.create(registerDto);
    const savedUser = await this.userRepository.save(user);

    return new UserResponseDto(savedUser);
  }

  async findById(id: number): Promise<UserResponseDto> {
    const user = await this.userRepository.findOne({ where: { id } });

    if (!user) {
      throw new NotFoundException("User not found");
    }

    return new UserResponseDto(user);
  }
}
```

**ポイント:**

- `@Injectable()`: DI コンテナに登録
- ビジネスロジックの検証
- Repository を使ってデータベース操作

---

### 3.4 Guard (ガード)

続いて Guard です。
Guard は基本的には認証認可などのチェックの役割があります。
ここで検証した内容で通らなかったら Controller に入る前に弾かれます。
私はこの Guard = 型チェックをする だと思っていたので、改めて意味を知って衝撃を受けました。

意味的には「このリクエストは処理してもいいか？」を判別するためのものです。

用途の例としては以下のものがあるようです。

| 用途               | 説明                               |
| ------------------ | ---------------------------------- |
| 認証・認可         | ログインしているか、権限があるか   |
| レート制限         | リクエストが多すぎないか           |
| メンテナンスモード | サービスが利用可能か               |
| 機能フラグ         | この機能を使えるユーザーか         |
| 営業時間           | 今アクセスして良い時間帯か         |
| 地域制限           | アクセス可能な地域からか           |
| 所有権チェック     | 自分のリソースにアクセスしているか |
| API バージョン     | サポートされているバージョンか     |

**役割:** 認証・認可のチェックなど

```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {
  canActivate(context: ExecutionContext) {
    // JWTトークンを検証
    return super.canActivate(context);
  }
}

// 使用例
@Controller("users")
@UseGuards(JwtAuthGuard) // ← ログイン必須
export class UserController {}
```

**ポイント:**

- コントローラー実行前にチェック
- `true`を返すと処理続行、`false`で拒否

---

### 3.5 DTO (Data Transfer Object)

DTO は基本的には IF の設定になります。
その中で、 `@IsEmail` といったアノテーションを使ってチェックを行います。
私はここで大きな勘違いをしていて、「DTO に定義さえしておけば勝手にチェックされる」と思っていたのですが実際には
「ここで定義した上で、Validation Pipe がそれを見てチェックを実行すると言うことでした。
なので、DTO のアノテーション = Pipe という認識は間違っていることを学びました。
global に DTO の定義を用いたバリデーションを行うには、main.ts にて以下のような実装をする必要があります。

```ts
app.useGlobalPipes(new ValidationPipe());
```

設定が必要とはいえ、これだけで勝手に DTO 定義を見てバリデーションかけてくれるの嬉しいですね！

**役割:** 入力データの形式定義とバリデーションの定義（バリデーションの実行は Pipe）

```typescript
export class CreateUserDto {
  @IsEmail()
  @Transform(({ value }) => value.toLowerCase().trim())
  email: string;

  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
  password: string;
}
```

**ポイント:**

- `class-validator`でバリデーション
- `class-transformer`でデータ変換
- ValidationPipe が自動的に検証

---

### 3.6 Pipe (パイプ)

そして、バリデーションの実行をしてくれる Pipe さんです。
平易なチェックは基本 DTO の定義を見て勝手に ValidationPipe さんが実行してくれるものでなんとかなりそうです。
どういう場合にカスタマイズが必要なのかは、特に調べていないので気になった方は調べてみてください 🙇

**役割:** データ変換・バリデーション

```typescript
// グローバル設定
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    transform: true,
  }),
);

// 個別使用
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  // "123" → 123 (文字列から数値に自動変換)
}
```

**ポイント:**

- ValidationPipe: DTO のバリデーション
- ParseIntPipe: 文字列 → 数値変換

---

### 3.7 Entity (エンティティ)

私自身、昔は DTO と Entity の違いがわかっていませんでした。
DTO は Request や Response の方を決めるもので、Entity はテーブルの項目定義そのものだったんですね。
要するに該当のテーブルではどういう項目を持っているか、プライマリキーは何か、ユニークかどうかみたいな情報を持っているものが Entity になります。
この情報を使って Repository が DB を操作するということですね。

また、単体のテーブルの情報だけでなくテーブル間のリレーションの情報も Entity には記載されていたりします。
ライフサイクルフックを用いて保存前後の処理も定義したりできるようです。ライフサイクルフックについては私は使ったことがないです。

**役割:** データベーステーブルの定義

```typescript
@Entity("users")
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  email: string;

  @Column()
  name: string;

  @Column({ select: false })
  password: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @OneToMany(() => Task, (task) => task.user)
  tasks: Task[];

  @BeforeInsert()
  @BeforeUpdate()
  async hashPassword() {
    if (this.password && !this.password.startsWith("$2b")) {
      this.password = await bcrypt.hash(this.password, 12);
    }
  }
}
```

**ポイント:**

- TypeORM がテーブルとマッピング
- リレーションシップを定義
- ライフサイクルフックで自動処理

---

### 3.10 Repository (リポジトリ)

そして最後に Repository です。
データベース操作を基本的に担当します。
今回私が作ったツールでは基本的な実装しかしなかったので typeorm にある機能で DB 操作をしていました。
とても簡単でした！
より複雑なクエリを実行したい場合は、カスタマイズした Repository を定義することも可能です。

**役割:** データベース操作

```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>
  ) {}

  async findAll(): Promise<User[]> {
    return this.userRepository.find();
  }

  async findOne(id: number): Promise<User> {
    return this.userRepository.findOne({ where: { id } });
  }

  async create(data: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(data);
    return this.userRepository.save(user);
  }

  async update(id: number, data: UpdateUserDto): Promise<User> {
    await this.userRepository.update(id, data);
    return this.findOne(id);
  }

  async delete(id: number): Promise<void> {
    await this.userRepository.delete(id);
  }
}
```

**主要メソッド:**

- `find()`: 複数件取得
- `findOne()`: 1 件取得
- `create()`: インスタンス作成
- `save()`: 保存
- `update()`: 更新
- `delete()`: 削除

---

## 4. まとめ

### 4.1 アーキテクチャの全体像

全体的な処理の流れは以下の通りでした！
今回、Middleware と Interceptor については触れませんでしたが、実際にはこんな感じで処理に含まれています。

```
Middleware → Guard → Interceptor → Pipe → Controller → Service → Repository → Database
```

### 4.2 各コンポーネントの責務

触れなかったところも含めた処理の流れの中で各コンポーネントの責務としては以下のようにまとめます。

| コンポーネント  | 責任               |
| --------------- | ------------------ |
| **Module**      | 機能のグループ化   |
| **Controller**  | ルーティング       |
| **Service**     | ビジネスロジック   |
| **Repository**  | データベース操作   |
| **Guard**       | 認証・認可         |
| **DTO**         | 入力バリデーション |
| **Pipe**        | データ変換         |
| **Interceptor** | レスポンス加工     |
| **Middleware**  | リクエスト前処理   |
| **Entity**      | テーブル定義       |

### 4.3 学習した感想

バックエンドの知識が全くない状態で NestJS の実装を眺めていたことはあったのですが、具体的な処理の流れだったり各コンポーネントの責務が明確にわかるようになって、自分のアプリでもバックエンドの処理が追えるようになりました！
今回触れなかった Middleware や Interceptor などの知識も含めてより深く学習して、フロントエンドもバックエンドも深く理解していけるようにこれからも学習しようと思いました！

また、今回の学習のように AI を壁打ち相手にひたすら質問し続けるのはとてもよかったです。
知らない言語でもこうやって学べばキャッチアップがスムーズにいきそうだなと思いました。（AI が得意な言語に限るのだろうか...?）

長々と書いた駄文を見ていただきありがとうございました！

うぐいすソリューションズアドベントカレンダー XX 日目、PM などを歴任されている猛者の長田さんになります！
皆様、乞うご期待！！

---

## 📚 参考リソース

- [NestJS 公式ドキュメント](https://docs.nestjs.com/)
- [TypeORM 公式ドキュメント](https://typeorm.io/)
- [class-validator](https://github.com/typestack/class-validator)
- [class-transformer](https://github.com/typestack/class-transformer)

---
