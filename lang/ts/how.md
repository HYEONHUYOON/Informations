# TypeScript 작성 규칙 (Production-Ready)

본 가이드는 실제 프로덕션 환경에서 안정성, 가독성, 유지보수성을 최우선으로 두고 작성되었습니다. 짧음보다 명확함, 추상화 남발보다 실용성, 런타임 안전을 보장하는 설계를 지향합니다.

- 팀 기본 원칙
  - 타입 안전 우선: `any` 금지, 가능한 `unknown`/좁히기 사용
  - 명확함 > 축약: 타입과 반환값을 “보이는 곳”에 남긴다
  - 불변성 우선: `readonly`/`as const` 적극 활용
  - 모듈 경계는 타입으로 설명: `import type`/`export type` 활용
  - UI/인터랙션 협업 고려: 이벤트/애니메이션/상태의 타입을 엄격하게 유지

---

## 1) 프로젝트 설정 (권장 기본값)

### tsconfig.json (권장)
```jsonc
{
  // 프로덕션 기준 엄격 설정
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx", // React 사용 시. Vue/Solid 등은 해당 프레임워크 설정으로 변경
    "strict": true,
    "noImplicitAny": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "useUnknownInCatchVariables": true,
    "allowJs": false,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "skipLibCheck": true,

    // 경로 별칭 (선택)
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@types/*": ["types/*"]
    }
  },
  "include": ["src", "types"]
}
```

### ESLint/Prettier (요지)
- `@typescript-eslint` + recommended 설정 사용
- 포맷은 Prettier로 일원화, ESLint는 룰링에 집중
- import 정렬은 도구 자동화(예: eslint-plugin-import 또는 Prettier import sort 플러그인)

```js
// .eslintrc.js (요지)
module.exports = {
  root: true,
  parser: "@typescript-eslint/parser",
  plugins: ["@typescript-eslint", "import"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:import/recommended",
    "plugin:import/typescript"
  ],
  rules: {
    "@typescript-eslint/explicit-function-return-type": ["warn", { allowExpressions: true }],
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/consistent-type-imports": ["warn", { prefer: "type-imports" }],
    "import/order": ["warn", { "newlines-between": "always" }]
  }
};
```

---

## 2) 파일/폴더 구조
- `src/` 아래 도메인(피처) 기준으로 구성: `src/features/<name>/...`
- 공용 타입은 `types/` 디렉터리로 분리, 외부로 노출되는 타입만 재export
- `index.ts`는 명시적 re-export만 수행 (와일드카드 내보내기 지양)

```ts
// src/features/user/index.ts
export { getUser, updateUser } from "./api"; // 명시적 내보내기
export type { User, UserId } from "@types/user"; // 타입은 type export
```

---

## 3) 네이밍 규칙
- 타입/인터페이스/열거형: PascalCase (`User`, `UserProfile`, `FetchState`)
- 변수/함수/메서드: camelCase (`fetchUser`, `userName`)
- 제네릭: 짧지만 의미 있는 접두사 사용 (`TItem`, `TResult`)
- 상수: `const` 우선, 환경 변수/매직 넘버는 UPPER_SNAKE_CASE
- 파일: kebab-case, 타입 전용 파일은 `*.types.ts` 허용

```ts
// 예시
type UserId = string; // 사용자 식별자
interface User { id: UserId; name: string }

function fetchUser(id: UserId): Promise<User> { /* ... */ return Promise.resolve({ id, name: "" }); }
```

---

## 4) 타입 선언: interface vs type
- 기본은 `type`을 선호 (유니온/교차/매핑 타입 등 다루기 쉬움)
- 확장/선언 병합이 필요한 공개 API 형태는 `interface`
- 같은 개념을 `type`과 `interface`로 중복 정의 금지

```ts
// interface가 이점인 경우 (확장, 선언 병합)
interface ApiError { code: string; message: string }
interface ApiError { cause?: unknown } // 선언 병합 가능

// type이 편리한 경우 (유니온/교차)
type FetchStatus = "idle" | "loading" | "success" | "error";
type WithTimestamps<T> = T & { createdAt: string; updatedAt: string };
```

---

## 5) any 금지, unknown 권장, 단언 최소화
- `any`는 타입 시스템을 우회하므로 금지
- 외부/동적 데이터는 `unknown`으로 받고, 사용자 정의 타입가드로 좁힌다
- `as` 단언은 마지막 수단. 대신 `satisfies`를 적극 사용

```ts
// 나쁨
const data: any = JSON.parse(input);

// 권장: unknown + 좁히기
const dataUnknown: unknown = JSON.parse(input);

function isUser(x: unknown): x is { id: string; name: string } {
  if (typeof x !== "object" || x === null) return false;
  const o = x as Record<string, unknown>;
  return typeof o.id === "string" && typeof o.name === "string";
}

if (isUser(dataUnknown)) {
  // 여기서부터 안전하게 사용 가능
  console.log(dataUnknown.id);
}

// satisfies: 구조 검증 + 추론 유지
const theme = {
  spacing: { xs: 4, sm: 8, md: 12 } as const,
} satisfies { spacing: Record<string, number> };
```

---

## 6) 함수/메서드 설계
- 공개 함수는 반환 타입 명시 필수
- 매개변수는 객체 1개로 합치고, `Optional`은 `?`로 표현 (undefined와의 유니온 지양)
- 좁히기/판별 유니온을 통해 분기 명확화

```ts
// 공개 함수: 반환 타입 명시
export function add(a: number, b: number): number {
  return a + b;
}

// 옵션은 객체 파라미터로 묶기
type FetchOptions = { cache?: boolean; signal?: AbortSignal };
export async function fetchJson<T>(url: string, opts: FetchOptions = {}): Promise<T> {
  const res = await fetch(url, { signal: opts.signal, cache: opts.cache ? "force-cache" : "no-store" });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>;
}

// 사용자 정의 타입 가드 예시
function hasId<T extends { id?: unknown }>(x: T): x is T & { id: string } {
  return typeof (x as any).id === "string";
}
```

---

## 7) null/undefined 처리
- 옵셔널 체이닝/널 병합 사용: `?.`, `??`
- Non-null 단언 `!`는 금지 (테스트/네로잉로 대체)

```ts
const title = doc?.meta?.title ?? "Untitled"; // 안전한 기본값
```

---

## 8) 불변성, 컬렉션
- 기본 `const` 사용, 객체/배열은 `Readonly`/`as const`로 보호
- 가변 메서드(`push`, `splice`) 지양, 불변 패턴 사용

```ts
const items = [1, 2, 3] as const; // 읽기 전용 리터럴

function append<T>(list: ReadonlyArray<T>, item: T): ReadonlyArray<T> {
  return [...list, item]; // 원본 불변 유지
}
```

---

## 9) 에러 처리
- 문자열 throw 금지, `Error` 파생만 사용
- 결과를 값으로 다루는 패턴 권장 (`Result` 타입)

```ts
// 단순 Result 타입
type Ok<T> = { ok: true; value: T };
type Err<E = unknown> = { ok: false; error: E };
export type Result<T, E = unknown> = Ok<T> | Err<E>;

export async function safeFetch<T>(url: string): Promise<Result<T>> {
  try {
    const data = await fetchJson<T>(url);
    return { ok: true, value: data };
  } catch (e) {
    return { ok: false, error: e };
  }
}
```

---

## 10) 비동기/동시성
- 반환 타입은 항상 `Promise<...>`로 명시
- 떠다니는 Promise 금지: 반드시 `await` 또는 핸들링
- 병렬은 `Promise.all`/`allSettled` 사용, 취소는 `AbortController`

```ts
async function loadAll() {
  const controller = new AbortController();
  const [a, b] = await Promise.all([
    fetchJson<A>("/a", { signal: controller.signal }),
    fetchJson<B>("/b", { signal: controller.signal })
  ]);
  return { a, b };
}
```

---

## 11) 모듈 경계와 타입 전용 임포트
- 타입만 필요할 때는 `import type` 사용
- 사이드 이펙트 없는 타입 재export는 `export type`

```ts
import type { User } from "@types/user"; // 런타임 번들에서 제거됨
export type { UserId } from "@types/user";
```

---

## 12) 유틸리티 타입 (표준 우선)
- 표준 `Pick`/`Omit`/`Partial`/`Required`/`Readonly`/`Record`/`NonNullable`/`ReturnType`/`Parameters` 우선 사용
- 필요 시 좁은 범위의 로컬 유틸리티 추가 정의

```ts
type WithId<T> = T & { id: string };
```

---

## 13) 프론트엔드/인터랙션 고려 사항
- 이벤트 타입은 구체적으로 표기 (예: `React.MouseEvent<HTMLButtonElement>`)
- 애니메이션/마이크로 인터랙션의 상태/키는 상수화하고 타입으로 보장
- 시간/거리/알파 값 등은 단위 주석 또는 브랜딩된 타입으로 명시

```ts
// React 예시 (주석은 한국어로 명확히)
// 버튼 hover/press 마이크로 인터랙션 상태
export type ButtonState = "idle" | "hover" | "pressed" | "disabled";

// 애니메이션 키를 상수로 고정하고 타입화
export const MotionKeys = {
  enter: "btn-enter",
  exit: "btn-exit"
} as const;
export type MotionKey = typeof MotionKeys[keyof typeof MotionKeys];

// 이벤트 타입 구체화
function onClick(e: React.MouseEvent<HTMLButtonElement>) {
  e.preventDefault();
}
```

---

## 14) 외부 데이터/스키마
- 런타임 검증 도입 권장: `zod` 등으로 파싱 후 `z.infer`로 타입 획득
- API 바운더리는 입력/출력 타입을 명확히, 내부 도메인 타입과 분리

```ts
// zod 예시
import { z } from "zod";
export const UserSchema = z.object({ id: z.string(), name: z.string() });
export type User = z.infer<typeof UserSchema>;

export async function getUserStrict(id: string): Promise<User> {
  const raw = await fetchJson<unknown>(`/api/users/${id}`);
  return UserSchema.parse(raw); // 런타임 검사로 안전 보장
}
```

---

## 15) 로그/디버깅
- `console.*`는 개발 중에만 사용, 프로덕션에서는 로깅 유틸을 통해 레벨/샘플링 관리
- 로그의 shape/type을 미리 정의하여 가독성 확보

```ts
type LogPayload = { event: string; meta?: Record<string, unknown> };
function logInfo(msg: string, payload?: LogPayload) { /* ... */ }
```

---

## 16) 안티 패턴 (피하기)
- 와일드카드 export: `export * from "..."` (API 표면 불명확)
- 광범위한 단언: `as any`, `as unknown as T`
- Non-null 단언 `!` 남용
- 공유 mutable 상태 (특히 UI 상태)

---

## 17) 빠른 체크리스트 (PR 전 확인)
- [ ] `any` 없는가? 필요 시 `unknown` + 좁히기 적용했는가?
- [ ] 공개 함수/모듈의 반환 타입이 명시되어 있는가?
- [ ] 외부 데이터는 런타임 검증(zod 등)을 통과하는가?
- [ ] 불변성 유지(`readonly`, `as const`)가 지켜졌는가?
- [ ] 타입 전용 임포트/익스포트를 사용했는가?
- [ ] 인터랙션(이벤트/애니메이션) 관련 값이 상수화+타입화 되어 있는가?
- [ ] ESLint/Prettier가 통과되는가?

---

## 18) 예외 허용 범위
- 테스트 코드, 스파이크(실험) 브랜치에서는 완화 가능하나 PR 시 본 규칙으로 정리
- 레거시 마이그레이션 중에는 범위를 좁혀 점진적 적용

---

## 부록: 팀 합의 옵션
- 프레임워크별 JSX 설정(React/Vue/Solid)
- import 정렬 도구 선정(ESLint vs Prettier 플러그인)
- 경로 별칭 범위(@/*, @types/* 등)

> 필요 시 `tsconfig.json`, `.eslintrc.js`, `prettier` 기본 템플릿도 함께 제공합니다. 팀 상황을 알려주세요.