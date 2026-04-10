# gen-service-type

주어진 API PATH를 기반으로 백엔드 Response DTO를 역추적하여 프론트엔드 공통 서비스 타입을 생성한다.

## 입력
`$ARGUMENTS` — API PATH (예: `/api/v1/products`, `/api/seller/orders/{orderId}`)

## 작업 순서

### 1. 백엔드 Controller 탐색
- `apps/petfriends-backend-seller/api/src/main/java/` 하위에서 `$ARGUMENTS`에 해당하는 `@RequestMapping` 또는 `@GetMapping`/`@PostMapping` 등이 있는 Controller 파일을 찾는다.
- Controller 메서드의 반환 타입(Response DTO)을 확인한다.

> ⚠️ **`@ApiResponse @Schema`를 실제 응답 타입으로 신뢰하지 말 것**
>
> 컨트롤러 반환 타입이 `ResponseEntity<?>`(와일드카드)인 경우, `@Schema(implementation = SomeDto.class)`는 Swagger 문서용 어노테이션일 뿐이며 실제 런타임 응답 타입과 다를 수 있다.
> 반드시 컨트롤러가 호출하는 **UseCase/Service 메서드의 실제 반환 타입 import**를 끝까지 추적한다.
>
> ```java
> // ❌ 이것만 보고 응답 타입을 결론 내리지 말 것
> @ApiResponse(content = { @Content(schema = @Schema(implementation = SomeFlatDto.class)) })
> public ResponseEntity<?> getProduct(...) {
>     return CommonResponse.setResponse(useCase.getProduct(id)); // ← 실제 타입은 여기
> }
> ```
>
> **추적 순서**: Controller → UseCase `import` 구문 → 실제 반환 DTO 클래스

> ⚠️ **같은 이름의 DTO가 여러 패키지에 존재할 수 있음**
>
> 예: `list/ProductDetailResponse`(Swagger 참조용 플랫 구조)와 `productdetail/ProductDetailResponse`(UseCase 실제 반환 중첩 구조)처럼 동일한 클래스명이 다른 패키지에 공존할 수 있다.
> 처음 발견한 것에서 탐색을 멈추지 말고, **UseCase의 `import` 구문에서 어느 패키지 것인지 반드시 확인**한다.

### 2. Response DTO 역추적 — 요청 DTO와 응답 DTO를 반드시 구분
- Controller가 반환하는 DTO 클래스를 찾아 모든 필드를 파악한다.
- 중첩된 DTO/Entity/Enum이 있으면 재귀적으로 추적한다.

> ⚠️ **요청(Request) DTO와 응답(Response) DTO를 혼동하지 말 것**
>
> 같은 도메인에 이름이 유사한 DTO가 request용/response용으로 따로 존재할 수 있다.
> 예: `ProductAttributeDto`(request — `productAttributeGroup2Id`만 포함)와 `OriginProductAttributeDto`(response — 표시용 name 필드 전체 포함).
>
> 프론트엔드 타입은 반드시 **응답 DTO** 기준으로 작성한다.
> 응답 DTO는 Controller → UseCase 반환 타입을 추적하면 찾을 수 있으며, `Origin*`, `*Response`, `*Dto` 등 네이밍이 다를 수 있으므로 클래스 파일을 직접 열어 필드를 확인한다.
>
> - `@JsonProperty`가 붙은 getter 메서드는 직렬화 시 JSON에 포함되므로 **가상 필드**로 타입에 반드시 추가한다.
> - `@JsonProperty` 가상 필드가 `String.format()` 등으로 항상 값을 반환하는 경우 `Nullable`이 아닌 `string`으로 선언한다.
- 다음 Java → TypeScript 직렬화 규칙을 적용한다:

| Java 타입 | TypeScript 타입 | 비고 |
|---|---|---|
| `LocalDate` | `string` | ISO 형식 `"YYYY-MM-DD"` |
| `LocalDateTime` | `string` | ISO 형식 `"YYYY-MM-DDTHH:mm:ss"` |
| `Enum` | string literal union | enum의 `name()` 기준, 예: `'SALE' \| 'STANDBY'` |
| `YNType` | `'Y' \| 'N'` | |
| `Integer` / `Long` | `number` | |
| `Boolean` | `boolean` | |
| `String` | `string` | |
| nullable/Optional 필드 | `T \| null` | |
| `List<E>` | `T[]` | |
| `BigDecimal` / `Double` | `number` | |

### 3. 도메인 분류
- API PATH에서 도메인을 추론한다 (예: `/products` → product, `/orders` → order).
- 대상 디렉토리: `apps/petfriends-internals-admin-frontend/apps/seller-admin/src/commons/services/{domain}/`

**파일 분리 규칙:**
- 도메인 namespace가 **하나**뿐이면 단일 파일 `{domain}.ts`로 생성한다.
- 도메인 namespace가 **둘 이상**이거나 파일이 너무 커지면 namespace별로 분리한다:
  - `{domain}/index.ts` — 주 namespace (예: `Product`) + 다른 파일 re-export
  - `{domain}/{namespace}.ts` — 부 namespace (예: `SellerRequest` → `request.ts`)
- `index.ts`는 분리된 파일을 `export { Foo } from './foo'`로 re-export하여 외부 import 경로(`@commons/services/{domain}`)가 변하지 않도록 유지한다.
- 부 namespace 파일의 타입을 주 namespace 파일에서 참조해야 할 경우 `import type { Foo } from './foo'`로 가져온다.

**현재 product 도메인 구조 (참고):**
```
src/commons/services/product/
├── index.ts    — Product namespace (Item, Detail 등)
└── request.ts  — SellerRequest namespace
```
`index.ts`에서 `export { SellerRequest } from './request'`로 re-export하므로  
외부에서는 `import { Product, SellerRequest } from '@commons/services/product'` 그대로 사용한다.

- 기존 파일이 있으면 기존 내용을 유지하면서 타입을 추가/병합한다.

### 4. 타입 생성 규칙

**Nullable 규칙:**
- `T | null` 형태는 반드시 `Nullable<T>`로 표기한다.
- 파일 상단에 `import type { Nullable } from '@teampetfriends/util-types';`를 추가한다.

```typescript
// ✅
import type { Nullable } from '@teampetfriends/util-types';
vendorId: Nullable<number>;

// ❌
vendorId: number | null;
```

**JSDoc 규칙:**
- 백엔드 DTO/Entity의 각 필드에 달린 주석(`//`, `/* */`, `/** */`, `@ApiModelProperty`, `@Schema`, Swagger 어노테이션 등)을 확인하고, 있으면 TypeScript 필드에 JSDoc(`/** */`)으로 기재한다.
- 백엔드에 주석이 없는 필드는 JSDoc 없이 그대로 정의한다. 임의로 설명을 만들어 넣지 않는다.

**구조 규칙 (엄수):**
- 서브 타입(`ProductItem`, `ProductDetail` 등)은 **파일 내 별도 타입으로 정의하되 `export`하지 않는다** (private).
- 네임스페이스 타입에서 `Item: ProductItem` 형태로 참조한다.
- 모든 키(타입 프로퍼티 및 서브 타입 필드)는 **알파벳 오름차순**으로 정렬한다.
- 각 필드 끝에 **세미콜론**을 붙인다.

**export 패턴 — type + namespace 병합 (declaration merging):**
- `export type DomainName` + `export namespace DomainName`을 함께 선언한다.
- `export type`은 실제 JSON 응답 구조를 그대로 표현한다 (프로퍼티 키는 JSON 필드명 기준 camelCase).
- `export namespace`는 도트 표기법(`Domain.SubType`) 접근을 위한 타입 별칭을 제공한다.
- namespace 안에서 서브 타입을 `export type`으로 노출하면 외부에서 `Domain.SubType` 또는 `Domain['SubType']` 둘 다 사용 가능하다.
- 서비스 파일 상단에 반드시 아래 주석을 추가한다 (설정 파일 수정 없이 namespace를 허용):
  ```typescript
  /* eslint-disable @typescript-eslint/no-namespace */
  ```

```typescript
// ✅ 올바른 구조
type ProductItem = {
  averageRate: number;
  sellingPrice: Nullable<number>;
};

// export type: 실제 JSON 응답 타이핑에 사용
export type Product = {
  Item: ProductItem;
  Status: 'SALE' | 'STANDBY' | 'PAUSE_SOLD_OUT' | 'STOP_SALE';
};

// export namespace: 도트 표기법 접근 지원
export namespace Product {
  export type Item = ProductItem;
  export type Status = 'SALE' | 'STANDBY' | 'PAUSE_SOLD_OUT' | 'STOP_SALE';
}

// 사용 측
const data: Product.Item = ...; // ✅ 도트 표기법
const data: Product['Item'] = ...; // ✅ 브라켓 표기법 (둘 다 가능)
```

**namespace 안에 서브 namespace가 필요한 경우 (깊은 서브 타입 노출):**
- `Register`처럼 프로퍼티가 많은 타입은 해당 프로퍼티 서브 타입도 namespace로 노출한다.

```typescript
export namespace SellerRequest {
  export type Register = SellerRequestRegister; // 전체 응답 타입
  export namespace Register {
    export type StandardDto = SellerRequestStandardDto; // 서브 타입
    export type CategoryDto = SellerRequestCategoryDto;
    // ...
  }
}
// 사용 측
SellerRequest.Register            // 전체 응답 타입
SellerRequest.Register.StandardDto // 서브 타입
```

**주의: JSON 필드명은 camelCase 유지**
- `export type`의 프로퍼티 키는 실제 API JSON 필드명과 일치해야 하므로 camelCase를 유지한다.
- namespace의 타입 별칭 이름(`StandardDto`, `CategoryDto` 등)은 PascalCase로 지어 타입임을 명확히 한다.

```typescript
// ❌ 잘못된 구조
export type SellerRequest = {
  Register: {
    ProductDetailStandardDto: ...; // JSON 필드명은 camelCase여야 함
  };
};
```

- 일회성/페이지 전용 Response 타입(파라미터, 리스트 래퍼 등)은 이 파일에 넣지 않는다.
- 이미 `src/commons/services/`에 동일한 타입이 있으면 중복 생성하지 않고 재사용을 안내한다.

### 5. 기존 지역 타입을 공통 타입으로 대체
새로 만든 공통 타입과 의미상 동일한 지역 타입이 존재하면 전면 대체한다.

**대체 작업 범위:**
- `pages/*/modules/interfaces.ts` — 지역 타입 정의 삭제
- `pages/*/apis/` — axios 제네릭 타입 교체 (`client().get<OldType>` → `client().get<Product.Item>`)
- `pages/*/modules/hooks/`, `pages/*/modules/components/` 등 — 해당 지역 타입을 참조하는 모든 파일의 타입 annotation 교체
- 교체 후 import 경로를 `@commons/services/{domain}`으로 업데이트하고, `import type` 키워드를 사용한다.

**대체 판단 기준:**
- 필드 구조가 실질적으로 동일하면 대체한다
- 지역 타입이 공통 타입보다 필드가 많거나 적으면(일부만 겹침) 대체하지 않고 사용자에게 확인을 요청한다

**예시:**
```typescript
// interfaces.ts — Before (삭제 대상)
type ProductInquireItem = {
  productId: number
  productName: string
  sellingPrice: number | null
}

// apis/index.ts — Before
import type { ProductInquireItem } from '../interfaces';
const { data } = await client(true).get<ProductInquireItem[]>('/products');

// apis/index.ts — After
import type { Product } from '@commons/services/product';
const { data } = await client(true).get<Product['Item'][]>('/products');
```

### 6. 결과 보고
작업 완료 후 아래 형식으로 요약한다:
- 탐색한 Controller / DTO 파일 경로
- 생성/수정한 파일 경로 목록
- 생성된 타입 목록
- JSON 직렬화 변환이 적용된 필드와 그 이유
- 기존 지역 타입에서 대체한 타입 목록
- 사용자 확인이 필요한 애매한 타입 (있을 경우)
