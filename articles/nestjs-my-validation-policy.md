---
title: "[廃棄]Nest.jsで私はこんな感じでValidationしました"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['nestjs', 'classvalidator', 'classtransformer']
published: false
---

### 確認したversions

## 最終的にこんな感じになります
バリデーションエラー時に帰ってくるJSON

```json
{
  "error": "Bad Request",
  "message": [
    "それぞれの銘柄IDは0以上の数字を指定してください",
    "表示件数は0以上の数字を指定してください"
  ],
  "statusCode": 400,
  "path": "/brands?ids=1,a&limit",
  "timestamp": "2020-11-11T18:02:17.092Z"
}
```

# おおまかな方針

- `class-transformer`, `class-validator`でDTOクラスを作ってバリデーション
- 日本語のエラーメッセージを返したいのでclass-validatorの各デコレータをラップしたデコレータを作る
- `BadRequestException`のExceptionFilterでいい感じにエラーレスポンスを整形

という公式Docのサンプルべったりな方針で進めました

## `class-transformer`, `class-validator`でDTOクラスを作ってバリデーション

```typescript
export const toNumberList = (value: string): Array<number> => {...};

export class SearchQueryDTO {
  @CIsPositive({
    propertyName: 'ID',
    validationOptions: { each: true },
  })
  @IsOptional()
  @Transform((value) => toNumberList(value))
  readonly ids?: number[];

  @CIsString({
    propertyName: '〇〇名',
  })
  @IsOptional()
  readonly name?: string;
}

export class AController {
  readonly #AService: AService;

  constructor(AService: AService) {
    this.#AService = AService;
  }

  async findAll(@Query() query: SearchQueryDTO): Promise<AEntity[]> {
    return await this.#AService.findWith(query);
  }
}
```

`CIsPositive`や`CIsString`でカスタムなバリデーションデコレータです
また`main.ts`で

```typescript
const classValidatorOption = {
  skipMissingProperties: true,
  whitelist: true,
  forbidNonWhitelisted: true,
  forbidUnknownValues: true,
  transform: true,
};
app.useGlobalPipes(
  new ValidationPipe(
    Object.assign(classValidatorOption, {
      exceptionFactory: (errors: unknown) => new BadRequestException(errors),
    }),
  ),
);
```

というようにすれば、DTOクラスに沿ったリクエストのみがアクションのメソッド内部に渡されるようになります
個人的に、バリデーションとリクエストパラメタの変形を一つのクラスで出来て俯瞰できるこの方法好きです

## 日本語のエラーメッセージを返したいのでclass-validatorの各デコレータをラップしたデコレータを作る

前段のコードで登場した`CIsPositive`や`CIsString`といったものですが、class-validatorのビルトインメソッドをラップしただけのものです

```typescript
export const CIsPositive = (option: CValidateOption): PropertyDecorator => {
  return IsPositive(
    buildValidationOptions(
      option,
      `${option.propertyName}は0以上の数字を指定してください`,
    ),
  );
};
```

再利用できるように自分なりの表現でカスタムデコレータとして再定義してあげると良いでしょう
いくつかのデコレータの実装サンプルの詳細は以下リンクを見ていただけばと思います

https://github.com/michihiko-karino/sakenowa-data-prj-rest-api/blob/master/src/decorators/standardClassValidators.ts

## `BadRequestException`のExceptionFilterでいい感じにエラーレスポンスを整形

Nest.jsには`ExceptionFilter`という発生したExceptionの発生に応じて処理をしたり、レスポンスを整形する機能があって便利です
