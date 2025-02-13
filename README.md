### 1. API 함수 구현 (`src/api/userApi.ts`)

`apiClient.ts`에 정의된 `apiClient`를 사용하여 API 함수를 구현합니다. `RequestConfig` 타입을 사용하여 요청 설정을 정의하고, `handleApiCall` 함수를 사용하여 에러를 처리합니다.

```typescript:src/api/userApi.ts
import { apiClient, RequestConfig } from "./apiClient";
import { handleApiCall } from "./errorHandler";
import { User } from "../types/user";

const userApi = {
  createUser: (userData: Omit<User, "id">): Promise<User> => {
    const config: RequestConfig<Omit<User, "id">> = {
      url: "/users",
      method: "post",
      data: userData,
    };
    return handleApiCall(apiClient.post<User>(config));
  },

  getUser: (id: number): Promise<User> => {
    const config: RequestConfig = {
      url: `/users/${id}`,
      method: "get",
    };
    return handleApiCall(apiClient.get<User>(config));
  },

  updateUser: (id: number, userData: Partial<User>): Promise<User> => {
    const config: RequestConfig<Partial<User>> = {
      url: `/users/${id}`,
      method: "put",
      data: userData,
    };
    return handleApiCall(apiClient.put<User>(config));
  },

  deleteUser: (id: number): Promise<void> => {
    const config: RequestConfig = {
      url: `/users/${id}`,
      method: "delete",
    };
    return handleApiCall(apiClient.delete<void>(config));
  },

  getUsers: (): Promise<User[]> => {
    const config: RequestConfig = {
      url: "/users",
      method: "get",
    };
    return handleApiCall(apiClient.get<User[]>(config));
  },
};

export default userApi;
```

**설명:**

- `RequestConfig` 타입을 사용하여 API 요청에 필요한 `url`, `method`, `data`, `params` 등을 설정합니다.
- `apiClient`의 `get`, `post`, `put`, `delete` 함수를 사용하여 API 요청을 보냅니다.
- `handleApiCall` 함수를 사용하여 API 호출에서 발생하는 에러를 처리합니다.

### 2. Query Hooks 구현 (`src/hooks/useUser.ts`)

`react-query`의 `useQuery`와 `useMutation` 훅을 사용하여 API 함수를 호출하고, 데이터를 캐싱합니다. `QUERY_KEYS`를 사용하여 쿼리 키를 관리하고, `queryClient.invalidateQueries`를 사용하여 캐시를 무효화합니다.

```typescript:src/hooks/useUser.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import userApi from "../api/userApi";
import { QUERY_KEYS } from "../api/queryKeys";
import { User } from "../types/user";

export const useUser = (id: number) => {
  const { data, error } = useQuery<User, Error>(
    QUERY_KEYS.user.detail(String(id)),
    () => userApi.getUser(id)
  );

  return { data, error };
};

export const useUsers = () => {
  const { data, error } = useQuery<User[], Error>(
    QUERY_KEYS.user.base,
    userApi.getUsers
  );

  return { data, error };
};

export const useCreateUser = () => {
  const queryClient = useQueryClient();

  return useMutation(userApi.createUser, {
    onSuccess: () => {
      queryClient.invalidateQueries(QUERY_KEYS.user.base);
    },
  });
};

export const useUpdateUser = (id: number) => {
  const queryClient = useQueryClient();

  return useMutation((userData: Partial<User>) => userApi.updateUser(id, userData), {
    onSuccess: () => {
      queryClient.invalidateQueries(QUERY_KEYS.user.detail(String(id)));
      queryClient.invalidateQueries(QUERY_KEYS.user.base);
    },
  });
};

export const useDeleteUser = (id: number) => {
  const queryClient = useQueryClient();

  return useMutation(() => userApi.deleteUser(id), {
    onSuccess: () => {
      queryClient.invalidateQueries(QUERY_KEYS.user.base);
    },
  });
};
```

**설명:**

- `useQuery` 훅을 사용하여 사용자 정보를 가져오는 쿼리를 정의합니다.
  - `QUERY_KEYS.user.detail(String(id))`를 쿼리 키로 사용합니다.
  - `userApi.getUser(id)`를 쿼리 함수로 사용합니다.
- `useMutation` 훅을 사용하여 사용자 정보를 생성, 수정, 삭제하는 뮤테이션을 정의합니다.
  - `userApi.createUser`, `userApi.updateUser`, `userApi.deleteUser`를 뮤테이션 함수로 사용합니다.
  - `onSuccess` 콜백 함수에서 `queryClient.invalidateQueries`를 호출하여 캐시를 무효화하고, 데이터를 다시 불러옵니다.

```tsx:src/components/UserDetail.tsx
import React from "react";
import { ErrorBoundary, FallbackProps } from "react-error-boundary";
import { useUser } from "../hooks/useUser"; // 사용자 데이터 조회 훅
import { User } from "../types/user"; // 사용자 타입 정의

function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen p-4 bg-gray-50">
      <h2 className="text-2xl text-red-500 font-bold mb-2">오류가 발생했습니다!</h2>
      <pre className="bg-gray-100 p-2 rounded text-sm text-gray-800">{error.message}</pre>
      <button
        onClick={resetErrorBoundary}
        className="mt-4 px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 transition"
      >
        다시 시도하기
      </button>
    </div>
  );
}

interface UserDetailProps {
  id: number;
}

function UserDetail({ id }: UserDetailProps) {
  const { data: user, error } = useUser(id);

  if (error) {
    throw error;
  }

  return (
    <div className="p-4 max-w-md mx-auto bg-white rounded shadow">
      <h2 className="text-xl font-bold mb-4">사용자 정보</h2>
      <p className="mb-2">
        <strong>이름:</strong> {user.name}
      </p>
      <p className="mb-2">
        <strong>이메일:</strong> {user.email}
      </p>
    </div>
  );
}

export function UserDetailWithBoundary({ id }: UserDetailProps) {
  return (
    // Loading은 별도의 스켈레톤Ui 구축 후 서스펜스로 위임
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <UserDetail id={id} />
    </ErrorBoundary>
  );
}
```

---

### 코드 설명

- **ErrorFallback 컴포넌트**

  - `react-error-boundary`에서 제공하는 `FallbackProps` 타입을 사용하여 에러 객체와 에러 리셋 함수를 받아옵니다.
  - Tailwind CSS 클래스를 사용하여 간단한 스타일의 fallback UI를 구현합니다.
  - 버튼을 클릭하면 `resetErrorBoundary` 함수가 호출되어 에러 경계를 초기화합니다.

- **UserDetail 컴포넌트**

  - `useUser` 훅을 사용하여 사용자 정보를 API로부터 가져옵니다.
  - 로딩 중에는 로딩 메시지를, 에러가 발생한 경우 에러를 throw하여 상위의 에러 바운더리로 전달합니다.
  - 정상적으로 데이터를 가져왔을 때 사용자 정보를 화면에 출력합니다.

- **UserDetailWithBoundary 컴포넌트**
  - `ErrorBoundary` 컴포넌트로 `UserDetail` 컴포넌트를 감싸 에러 발생 시 `ErrorFallback`이 렌더링되도록 합니다.
