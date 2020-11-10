---
title: "Nest.jsで@Paramと@Queryを1つのDTOクラスでTransformとValidateするTIPS"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['nestjs', 'classvalidator', 'classtransformer']
published: true
---

### 確認したversions

```
@nestjs/common: 7.4.2
class-transformer: 0.3.1
class-validator: 0.12.2
また以下両方で動作確認済
platform-express version : 7.4.4
platform-fastify version : 7.4.4
```

# やりたいこと

`/model/1?query=something`というリクエストを一つのDTOファイルでTransformとValidateをかけたい！

```typescript
class GetQueryDTO {
  @IsInt()
  @Transform((value) => Number(value))
  readonly id!: number;

  @IsString()
  @IsOptional()
  readonly query?: string;
}

---

@Get(':id')
async findAll(@???() params: GetQueryDTO): Promise<Entity[]> {
  return await this.#Service.findWith(params);
}
```

あくまでも幾つものある方法のひとつのやり方でしかありませんが

# `:id`というキー名の決め打ちのParamとQuery全部を引き受けるparam decoratorを適用して一つのリクエストオブジェクトにする

```ts
export const IdAndQuery = createParamDecorator(
  async (_data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    // `id`というキーを決め打ちで取得する
    const id = request.params.id;

    return { id, ...request.query };
  },
);
```

またTransformとValidateするPipeも定義します
コードは[公式DocのPipeの章](https://docs.nestjs.com/pipes#class-validator)の`validation.pipe.ts`を参考にしました

```ts
@Injectable()
export class ValidateAndTransformPipe
  implements PipeTransform<{ [key: string]: string }> {
  async transform(
    value: { [key: string]: string },
    { metatype }: ArgumentMetadata,
  ): Promise<any> {
    if (!metatype) {
      return value;
    }
    const transformed = plainToClass(metatype, value);
    // classValidatorOptionにはお好きなバリデーションオプションを渡してください
    const errors = await validate(transformed, classValidatorOption);
    if (errors.length > 0) {
      throw new BadRequestException(errors);
    }
    return transformed;
  }
}
```

上記2つのClassを利用することで1ファイルでTransformとValidateすることができました

```typescript
async findOne(
  @IdAndQuery(ValidateAndTransformPipe) params: GetQueryDTO,
): Promise<Entity | void> {
  // paramsは{ id: number, query?: string }のオブジェクトになってる！
  return await this.#Service.findWith(params);
}
```

以上
