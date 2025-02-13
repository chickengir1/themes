# 에러 핸들링 절차 문서

이 문서는 본 프로젝트에서 사용하는 에러 핸들링 전략과 절차를 상세하게 설명합니다.
주요 내용은 다음과 같습니다.

- **인터셉터를 통한 에러 전역 처리**
- **API 호출 시 에러 핸들링 유틸리티 함수 (`handleApiCall`)**
- **React Query와 ErrorBoundary를 통한 컴포넌트 레벨 에러 처리**
- **실제 사용 예제와 흐름**

---

## 1. 인터셉터를 통한 에러 전역 처리

`src/api/interceptors.ts` 파일에서는 Axios 인스턴스에 등록되는 요청/응답 인터셉터를 정의합니다.
인터셉터는 모든 API 요청 전후에 공통 작업(예: 로깅, 토큰 첨부, 에러 핸들링 등)을 처리합니다.

- **요청 인터셉터**
  - API 요청 전에 필요한 헤더 추가 및 공통 설정을 적용합니다.
- **응답 인터셉터**
  - 정상 응답은 그대로 전달하고, 에러 발생 시 `responseErrorInterceptor`를 통해 에러를 전파합니다.

> **참고:**
> 실제 프로젝트에서는 인터셉터 내에서 에러 발생 시 사용자에게 추가 안내나 자동 재시도 로직 등을 구현할 수 있습니다.

---

## 2. API 호출 시 에러 핸들링 유틸리티 (`handleApiCall`)

`src/api/apiClient.ts` 파일의 API 함수들은 모두 `.then()` 체인을 사용하여 데이터를 추출합니다.
각 API 모듈(예: `src/api/userApi.ts`)에서는 호출 후 `handleApiCall` 함수를 사용하여 **에러 처리 절차를 일원화**합니다.

아래는 `handleApiCall` 함수의 예제 코드입니다.

```typescript:src/api/errorHandler.ts
import axios, { AxiosError } from "axios";

/**
 * API 호출 결과를 처리하는 유틸리티 함수입니다.
 * - 정상적인 경우 데이터를 그대로 반환합니다.
 * - 에러 발생 시, axios 에러 여부를 판별하여 구체적인 에러 메시지를 추출한 후 에러를 재전파합니다.
 *
 * @param apiPromise - API 호출을 나타내는 Promise
 * @returns API 응답 데이터
 * @throws 에러 메시지가 포함된 Error 객체
 */
export function handleApiCall<T>(apiPromise: Promise<T>): Promise<T> {
  return apiPromise
    .then((data) => data)
    .catch((error: unknown) => {
      // axios 에러 여부 확인
      if (axios.isAxiosError(error)) {
        const axiosError = error as AxiosError;
        // 서버에서 전달한 구체적인 에러 메시지 또는 기본 메시지 사용
        const message = axiosError.response?.data?.message || axiosError.message;
        console.error("API 호출 에러:", message);
        throw new Error(message);
      }
      console.error("API 호출 에러:", error);
      throw error;
    });
}
```

**핵심 포인트:**

- axios 에러인 경우, 서버로부터 전달된 상세 메시지를 추출하여 개발자 및 사용자에게 유의미한 정보를 전달합니다.

---

## 3. React Query와 ErrorBoundary를 통한 에러 처리 흐름

프로젝트에서는 [Tanstack React Query](https://tanstack.com/query/latest)와 [react-error-boundary](https://github.com/bvaughn/react-error-boundary)를 사용하여 **컴포넌트 레벨**에서 발생하는 에러를 처리합니다.

1. **React Query**

   - `queryClient` 설정(`src/api/queryClient.ts`)에서 `throwOnError: true` 옵션을 사용하여, 쿼리 혹은 뮤테이션에서 발생한 에러가 Promise를 reject하고,
   - 각 `useQuery` 혹은 `useMutation` 훅 내에서 에러 발생 시 이를 그대로 전파하도록 구성합니다.

2. **컴포넌트 내 에러 전파**

   - 예를 들어, `src/hooks/useUser.ts`의 `useUser` 훅에서 API 호출 에러가 발생하면 에러가 throw되고,
   - 이를 사용하는 컴포넌트(`UserDetail`)는 별도의 catch 없이 에러를 그대로 throw합니다.

3. **ErrorBoundary**
   - 상위 컴포넌트에서는 `ErrorBoundary` 컴포넌트로 감싸서 에러가 발생했을 때 fallback UI를 렌더링하도록 합니다.
   - [`UserDetailWithBoundary`](src/components/UserDetail.tsx) 컴포넌트가 그 예로, 에러 발생 시 `ErrorFallback` 컴포넌트를 통해 사용자에게 에러 안내를 표시합니다.

---

## 4. 전체 에러 흐름 요약

1. **API 요청 단계**
   - API 호출은 `apiClient`를 통해 이루어지며, 요청 전/후 인터셉터가 공통 작업(헤더, 로깅, 에러 전파 등)을 수행합니다.
2. **에러 핸들링 단계**
   - API 함수는 `.then()` 체인을 사용하여 결과를 처리한 후, `handleApiCall` 함수를 통해 에러를 통합 처리합니다.
   - axios 에러의 경우 서버에서 전달된 구체적 메시지를 사용하여 Error 객체를 재발생시킵니다.
3. **React Query 통합**
   - React Query의 `throwOnError` 옵션 덕분에 에러가 컴포넌트로 전파됩니다.
4. **UI 에러 처리 단계**
   - 컴포넌트 내에서 발생한 에러는 `ErrorBoundary`로 포착되어, 지정된 fallback UI(`ErrorFallback`)가 렌더링됩니다.
5. **사용자 재시도**
   - fallback UI에서는 사용자가 에러를 확인하고 다시 시도할 수 있도록 `resetErrorBoundary` 함수를 제공합니다.
