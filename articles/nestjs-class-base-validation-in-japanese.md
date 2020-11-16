---
title: "[Nest.js]classベースなバリデーションで日本語エラーメッセージ返したい"
emoji: "🗾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['nestjs', 'classvalidator', 'classtransformer']
published: false
---

### 確認したversions

```
class-transformer: 0.3.1
class-validator: 0.12.2
```

# やりたいこと

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

上記の例のように、バリデーションエラーメッセージを日本語にします

# 前置き：Nest.jsのバリデーション

https://docs.nestjs.com/techniques/validation

Nest.jsのバリデーションとして公式で推奨されている方法は、DTO(data transfer object)classを定義してメンバーにデコレータを付与する方法です

```ts:id.dto.ts
import { Transform } from 'class-transformer';
import { IsInt } from 'class-validator';

export class IdDto {
  @IsInt()
  @Transform((value) => Number(value))
  readonly id!: number;
}
```

`class-validator`, `class-transformer`という同じ作者の２つのライブラリに依存しており、受け取ったパラメータをどう変換するか、何を正しいとするかをカスタマイズしやすく、協調させて動作させやすいです

```json:得られるレスポンス
{
  "statusCode": 400,
  "message": [
    "id must be an integer number"
  ],
  "error": "Bad Request"
}
```

バリデーションエラーメッセージをどう見せたいかは議論の余地はあると思いますが、ひとまず日本人ユーザが見ても分かりやすいメッセージを返したいです。簡単な方法として以下のような方法があります

# class-validatorのデコレータをラップした独自デコレータを使う

https://github.com/typestack/class-validator/blob/develop/src/decorator/ValidationOptions.ts

IsIntなどの各バリデーションデコレータには共通のオプションを渡せます
メッセージが英語のままでいいのであれば、ここらへんのオプションを適切に指定してあげれば大丈夫ですが…

```ts
// propertyName：メッセージ上での表示名
// errorMessage：表示されるメッセージを直接指定する
// validationOptions：デコレータに渡すValidationOptions
export type CValidateOption = {
  propertyName: string;
  errorMessage?: string;
  validationOptions?: ValidationOptions;
};

// ValidationOptionsのmessageを組み立てるメソッド
const buildValidationOptions = (
  option: CValidateOption,
  defaultMessage: string,
): ValidationOptions => {
  let validationOptions: ValidationOptions = {};
  let prefixMessage = '';

  if (option.validationOptions) validationOptions = option.validationOptions;
  // ValidationOptionsのeach(配列状のパラメタそれぞれに対してバリデーションをかけるオプション)が有効であれば
  // プロパティ名の前に「それぞれ」を付与するようにしている
  if (validationOptions.each) prefixMessage = 'それぞれの';

  validationOptions.message = option.errorMessage
    ? option.errorMessage
    : `${prefixMessage}${defaultMessage}`;
  return validationOptions;
};

export const CIsInt = (option: CValidateOption): PropertyDecorator => {
  return IsInt(
    buildValidationOptions(
      option,
      `${option.propertyName}は数値を指定してください`,
    ),
  );
};
```

デコレータを返すデコレータを作れば、既存のデコレータと同じように使うとこができます

```ts:id.dto.ts
export class IdDto {
- @IsInt()
+ @CIsInt({ propertyName: 'ID' })
  @Transform((value) => Number(value))
  readonly id!: number;
}

// 
// "message": [
//   "IDは数値を指定してください"
// ],
```

## サンプル実装
IsInt以外のデコレータも必要に応じて追加しています。何かのお役に立てれば幸いです

https://github.com/michihiko-karino/sakenowa-data-prj-rest-api/blob/master/src/decorators/standardClassValidators.ts

以上
