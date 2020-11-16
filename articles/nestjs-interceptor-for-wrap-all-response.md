---
title: "[Nest.js]REST API全エンドポイントで帰属表示を追加したかった時にしたこと"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['nestjs', 'classvalidator', 'classtransformer']
published: true
---

# やりたかったこと

帰属表示が必要な公開されているデータを使ったREST API実装を作った
公開するにあたりAPIのレスポンスに帰属表示を追加したかった
あとSwaggerドキュメント上のレスポンス例にも帰属表示を含めた形にした）

```json
{
  "description_ja": "このデータはさけのわによって提供されています。",
  "description_en": "This data is provided by sakenowa.",
  "sakenowa_link": "https://sakenowa.com",
  "data": {
    ...それぞれのレスポンス
  }
}
```

- swagger ui上での表示

![](https://storage.googleapis.com/zenn-user-upload/tb8hh9gq628ksr3uynraqzcou4qg)


# 具体的な方法と実装

https://github.com/michihiko-karino/sakenowa-data-prj-rest-api

さけのわデータプロジェクトで提供されているデータを取得し提供するREST API実装です


## Interceptorを使ってレスポンスを変形させる

https://docs.nestjs.com/interceptors

NestjsのInterceptorはリクエストの前後にロジックを追加したり、レスポンスを加工したりできます

```ts:license.interceptor.ts
export const License = {
  description_ja: 'このデータはさけのわによって提供されています。',
  description_en: 'This data is provided by sakenowa.',
  sakenowa_link: 'https://sakenowa.com',
};

export type Response<T> = typeof License & {
  data: T;
};

@Injectable()
export class LicenseInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<Response<T>> {
    return next
      .handle()
      // dataというプロパティがエンドポイントの返却値そのものになる
      .pipe(map((data) => Object.assign({}, License, { data }))); 
  }
}
```

```ts
export class BrandsController {
  readonly #brandsService: BrandsService;

  @Get()
  @UseInterceptors(LicenseInterceptor)
  async findAll(@Query() query: SearchQueryDTO): Promise<BrandEntity[]> {
    return await this.#brandsService.findWith(query);
  }
}
```

若干迷ったのは、メソッドの返却値を帰属表示を含めた型にするかどうかです
変形された型にキャストしてreturnすることで`findAll`の返却値が厳密になりますが、本質的ではないと考え変形前の型としました

## Swaggerの対応

Nestjsでは`ApiProperty`デコレータでアノテーションされたエンティティクラスを準備することで、簡単にswaggerドキュメントを生成できます

```ts
export class Area extends AreaEntity {
  @ApiProperty({ example: '5' })
  id: number;

  @ApiProperty({ example: '秋田県' })
  name: string;
}

import { ApiOkResponse } from '@nestjs/swagger/dist';

export class AreasController {
  @Get()
  @ApiOkResponse({ type: [Area] })
  @UseInterceptors(LicenseInterceptor)
  async list(): Promise<AreaEntity[]> {
    return this.#AreaRepository.find();
  }
}
```

ただし`LicenseInterceptor`による変形が考慮されたSwaggerドキュメントにするために工夫が必要です
Swagger上では実際のレスポンスに則した形を例示したかったため対応しました

```ts:licensedDTO.decorator.ts
export class LicensedDTO {
  @ApiProperty({ example: 'このデータはさけのわによって提供されています。' })
  description_ja: 'このデータはさけのわによって提供されています。';

  @ApiProperty({ example: 'This data is provided by sakenowa.' })
  description_en: 'This data is provided by sakenowa.';

  @ApiProperty({ example: 'https://sakenowa.com' })
  sakenowa_link: 'https://sakenowa.com';
}

export const LicensedDTODecorator = <TModel extends any>(model: TModel) => {
  const item = Array.isArray(model)
    ? {
        type: 'array',
        items: { $ref: getSchemaPath(model[0]) },
      }
    : {
        $ref: getSchemaPath(model as Type<any>),
      };

  return applyDecorators(
    ApiOkResponse({
      schema: {
        allOf: [
          { $ref: getSchemaPath(LicensedDTO) },
          {
            properties: {
              data: item,
            },
          },
        ],
      },
    }),
  );
};
```

swagger allofについてはこちら
https://swagger.io/docs/specification/data-models/oneof-anyof-allof-not/#allof

返却したいエンティティが単体(`Area`)なのか配列状(`[Area]`)なのかを指定できるようにしました
anyを使っているのはご容赦ください🙇‍♂️

```ts
export class AreasController {
  @Get()
- @ApiOkResponse({ type: [Area] })
+ @LicensedDTODecorator([Area])
  @UseInterceptors(LicenseInterceptor)
  async list(): Promise<AreaEntity[]> {
    return this.#AreaRepository.find();
  }
}
```

# おわりに

デコレータによる処理の共通化などには好みがありますが、個人的には整理されていれば一番読みやすく管理しやすいと思います
帰属表示が必要なREST APIというのは初めてでしたが特に詰まったりせず進められたので、改めてNestjsいいじゃんという気持ちです

以上
