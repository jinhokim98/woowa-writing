# UX를 향상시키는 서버 상태 관리 (react query v5)

## 서버 상태 관리의 필요성

프론트엔드 개발에서 상태 관리는 애플리케이션의 데이터 흐름과 사용자 경험(UX)에 직접적으로 영향을 미치는 중요한 요소입니다. 상태는 크게 클라이언트 상태와 서버 상태로 나눌 수 있으며, 이 중 서버 상태 관리는 신뢰성 있는 사용자 경험을 유지하기 위해 최신 데이터를 적시에 제공하는 데 필수적입니다.

서버 상태는 서버에서 가져오는 데이터를 의미하며, 사용자는 이를 통해 중요한 결정을 내리게 됩니다. 서버 상태가 신속하게 업데이트되면 사용자에게 최신 정보를 제공하여 서비스의 신뢰성을 높일 수 있습니다. 반면, 서버 상태를 정확하게 관리하지 못하면 사용자에게 잘못된 정보가 표시되거나 오래된 데이터가 유지될 수 있어 혼란을 초래할 수 있습니다.

이번 글에서는 이러한 서버 상태를 효과적으로 관리하기 위해 React Query를 선택한 이유와 함께 활용 방안을 소개하겠습니다. React Query는 서버 데이터의 캐싱, 동기화, 자동 갱신을 간편하게 설정할 수 있어 프론트엔드의 상태 관리 복잡성을 줄이고 사용자 경험을 개선하는 데 탁월한 도구입니다.

## 1. 서버 상태 변화에 따른 캐시 데이터 관리

사용자가 입력을 하고 전송을 누르면, POST, PUT, PATCH 등의 HTTP 요청이 서버로 전송됩니다. 이러한 요청은 서버 측에서 데이터베이스의 수정, 추가, 삭제와 같은 상태 변화를 일으킵니다. 문제는 클라이언트 측에서 캐시된 데이터가 이러한 서버 상태 변화를 인지하지 못하면, 여전히 변하기 전의 데이터를 보여줄 수 있다는 점입니다. 이는 사용자가 최신 데이터를 확인하지 못하거나 잘못된 정보를 보게 되어 혼란을 초래할 수 있습니다.

따라서 서버의 상태가 변하면, 클라이언트는 이를 감지하고 캐시된 데이터를 업데이트하거나 무효화해야 사용자에게 올바른 정보를 제공할 수 있게 됩니다. 서버 상태가 변했음을 클라이언트가 인지하는 방법에는 여러 가지가 있지만, 대표적으로 사용되는 방법 중 하나가 캐시 무효화(Cache Invalidation) 전략입니다.

### 캐시 무효화 전략

서버에 변화가 일어난 후, 클라이언트 측에서 해당 변화와 관련된 캐시 데이터를 무효화하여 다시 서버로부터 최신 데이터를 가져오는 방식입니다. 이 전략을 사용하면 클라이언트는 오래된 데이터를 계속 유지하지 않고, 서버의 최신 상태를 반영한 데이터를 사용자에게 제공합니다. 캐시 무효화는 주로 데이터 생성, 수정, 삭제 등의 요청이 성공한 이후에 수행되며 사용자는 실시간으로 갱신된 데이터를 확인할 수 있습니다.

```tsx
const queryClient = useQueryClient();

const { mutate } = useMutation({
  mutationFn: createCrew,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: [QUERY_KEYS.crewList] });
  },
});
```

useMutation 훅을 사용해 mutate 함수를 실행시킬 때, 요청이 성공하면 queryClient.invalidateQueries를 사용해 queryKey에 대한 캐시를 무효화하고 다시 데이터를 불러오게 됩니다.

이 방법은 가장 익숙한 방식이며, React Query로 서버 상태를 관리 할 때 자주 사용되는 전략입니다. 하지만 invalidateQueries 메서드를 사용할 때 아래 영상과 같은 현상을 마주했을 수 있습니다.

<img src="./image/invalidate_before.gif" width="200" />

#### 캐시 무효화의 한계와 개선

캐시를 무효화하는 과정에서 가끔 이전 데이터가 잠깐 보이는 현상이 발생할 수 있습니다. 이는 invalidateQueries가 현재 마운트된 컴포넌트의 활성 데이터만 즉시 업데이트하기 때문에 비활성화된(inactive) 데이터는 즉시 갱신되지 않기 때문입니다. 이로 인해 사용자에게 불안정한 UX를 제공할 수 있습니다.

이런 상황에서는 새로운 데이터를 불러오기 위해 invalidateQueries 옵션에서 refetchType을 'inactive'로 설정하는 것이 효과적입니다. refetchType은 무효화 시 어떤 상태의 캐시 데이터를 다시 가져올지 결정하는 옵션으로, 'none', 'active', 'inactive', 'all'을 선택할 수 있습니다. 이렇게 설정하면 현재 화면에 보이지 않는 데이터(inactive)라도 무효화 시 즉시 서버로 요청을 보내 업데이트할 수 있습니다.

```tsx
queryClient.invalidateQueries({
  queryKey: [QUERY_KEYS.crewList],
  refetchType: "inactive",
});
```

또 다른 방식으로는 refetchQueries를 사용하여 즉시 서버로부터 데이터를 다시 가져오게 하는 방법이 있습니다. 이렇게 inactive한 데이터를 다시 요청함으로써 사용자에게 더 정확한 데이터를 빠르게 보여줄 수 있습니다.

```tsx
queryClient.refetchQueries({
  queryKey: [QUERY_KEYS.crewList],
});
```

<img src="./image/invalidate_after.gif" width="500" />

하지만 이를 도입할 때 고려해야 할 점이 있습니다. 화면에 보이지 않는 데이터(inactive)를 다시 fetch하는만큼, 실제로 해당 데이터를 사용자가 얼마나 자주 접속하는지 확인해야 합니다. 서버로부터 새로운 정보를 미리 가져왔지만, 막상 사용자가 변경된 데이터를 잘 확인하지 않는다면 불필요한 요청일 수 있으니, 이를 적용할 때는 데이터의 중요도와 확인 빈도를 고려하여 적절히 도입해야 합니다.

### 낙관적 업데이트를 이용한 서버 상태 변화 관리

서버로의 요청이 필요하고 데이터 저장이 필요하지만, 결과의 정확도가 크게 중요하지 않은 경우(예: SNS 서비스의 '좋아요' 기능 등)에는 사용자에게 빠르게 피드백을 주는 것이 더 나은 경험을 줄 수 있습니다. 이런 간단한 작업에서 요청할 때마다 화면에 "로딩 스피너"(작업이 진행 중임을 나타내는 회전형 아이콘)가 나타나면 사용자 경험을 저하시킬 수 있습니다.

이럴 때 유용한 기법이 '낙관적 업데이트'입니다. 요청의 성공 여부와 관계없이, 사용자가 수행한 행동의 결과를 먼저 화면에 반영하여 빠른 피드백을 제공하는 방식입니다. 이를 위해 useMutation 훅을 사용해 mutate 함수를 실행하고, onMutate, onError, onSettled 등의 콜백을 통해 상태 변화를 제어할 수 있습니다.

```tsx
onMutate: async (data) => {
  const previousData = queryClient.getQueryData([data]);
  queryClient.setQueryData(["data"], (old) => [...old, newData]);

  return { previousData };
};
```

먼저 `onMutate` 훅에서는 `getQueryData`를 통해 이전 데이터를 저장해 둡니다. 이는 요청이 실패했을 때 원래 상태로 되돌리기 위해 필요한 데이터입니다. 이후 `setQueryData`를 사용하여, 요청이 성공했을 때 예상되는 상태로 캐시 데이터를 업데이트합니다. 마지막으로 `previousData`를 반환하여 이후 단계에서 사용할 수 있게 합니다.

```tsx
onError: (err, newData, context) => {
  queryClient.setQueryData(["data"], context.previousData);
};
```

`onError`는 요청이 실패했을 때 `onMutate`에서 저장해 두었던 `previousData`를 사용하여 변경된 데이터를 원래 상태로 복원합니다.

```tsx
onSettled: () => {
  queryClient.invalidateQueries(["data"]);
};
```

`onSettled`는 요청이 성공하거나 실패한 이후에 항상 실행되는 콜백입니다. 이 콜백을 통해 캐시 데이터를 무효화하고 서버에서 최신 데이터를 다시 가져오도록 하여, 요청 결과에 관계없이 데이터의 정확성을 유지할 수 있습니다. 이를 통해 낙관적 업데이트로 임시 적용했던 데이터가 실제 서버 데이터와 일치하는지 확인하게 됩니다.

지금까지 사용자가 서버에 데이터를 변경 요청할 때의 경험을 살펴보았다면, 이제는 서버로부터 데이터를 가져오는 GET 요청에서 더 나은 사용자 경험을 제공할 방법을 알아보겠습니다.

## 2. 서버 상태 불러오기

### Suspense로 인한 요청의 waterfall 현상 개선하기

Suspense 컴포넌트를 사용하면 데이터를 기다리는 동안 빈 화면 대신 Fallback을 보여줄 수 있습니다. 그러나 계층 구조에서 부모 컴포넌트와 자식 컴포넌트가 각각 API 요청을 한다면, 요청 처리 시 시간이 누적되어 지연이 발생할 수 있습니다. 이를 코드로 설명해 보겠습니다.

부모와 자식을 각각 1초간 API 요청을 기다리는 상황을 가정해 보겠습니다.

```tsx
 http.get(`${BASE_URL}/api/parent`, async () => {
    await new Promise(resolve => setTimeout(resolve, 1000));
    return HttpResponse.json(parent);
  }),

  http.get(`${BASE_URL}/api/child`, async () => {
    await new Promise(resolve => setTimeout(resolve, 1000));
    return HttpResponse.json(child);
  }),
```

부모 컴포넌트에 Suspense를 감싸 요청을 기다리면서 Fallback을 보여줍니다.

```tsx
<Suspense fallback={<Fallback text="부모를 불러오는 중" />}>
  <Parents />
</Suspense>
```

그리고 Parents 컴포넌트 내에서 자식을 불러옵니다.

```tsx
<Suspense fallback={<Fallback text="자식을 불러오는 중" />}>
  <Child />
</Suspense>
```

이렇게 됐을 때 부모와 자식은 각각 어떻게 보이게 될까요?

<img src="./image/waterfall_before.gif" />

위 코드에서는 부모의 데이터를 모두 받아온 후에야 자식 컴포넌트의 데이터를 요청하게 되며, 추가로 1초가 소요됩니다. 이를 'Suspense Waterfall 현상'이라고 합니다. 이 현상이 발생하는 이유는 Suspense가 부모 컴포넌트를 기다리는 동안 자식 컴포넌트를 불러오는 작업이 차단되기 때문입니다. 즉, 부모의 Suspense가 데이터를 다 가져올 때까지 자식 컴포넌트는 대기 상태에 놓여 렌더링되지 않고, 그 후에 자식의 데이터 요청이 시작됩니다. 따라서 부모와 자식 각각의 대기 시간이 누적되어 최종적으로 2초가 소요됩니다.

이 문제를 해결하려면 부모와 자식의 요청을 병렬로 처리해야 합니다. React Query에서는 useQueries 훅을 사용하여 이를 간단히 해결할 수 있습니다.

```tsx
const [parent, child] = useSuspenseQueries({
  queries: [
    {
      queryKey: [QUERY_KEYS.parent],
      queryFn: getParent,
    },
    {
      queryKey: [QUERY_KEYS.child],
      queryFn: getChild,
    },
  ],
});
```

부모와 자식 컴포넌트에서 각각 데이터를 가져오는 대신, useSuspenseQueries를 사용하여 데이터를 병렬로 요청하게 하면 아래 영상과 같이 Waterfall 현상을 해결할 수 있습니다.

<img src="./image/waterfall_after.gif" />

### 서버 상태를 미리 불러오기

서버 상태를 미리 불러옴으로써 사용자에게 로딩 화면을 보여주지 않고, 즉시 데이터를 제공할 수 있습니다. 특히 사용자가 곧바로 다음 화면으로 이동할 확률이 높을 때, 이 기능은 UX를 크게 개선할 수 있습니다.

페이지네이션이 대표적인 사례입니다. 사용자가 다음 페이지를 클릭할 가능성이 높을 때, 다음 페이지 데이터를 미리 캐시해둔다면 사용자에게 로딩 화면을 보여주지 않고 즉시 다음 페이지의 내용을 확인할 수 있게되고 빠르고 매끄러운 UX를 제공할 수 있습니다.

#### queryClient.prefetchQuery

React Query에서 queryClient.prefetchQuery 메서드를 사용하면 특정 데이터를 미리 불러와서 캐시에 저장할 수 있습니다. 첫 번째 인자는 query의 Key, 두 번째 인자는 데이터를 미리 불러올 콜백 함수입니다. 이 메서드를 호출하면 지정된 콜백이 실행되어 데이터를 캐싱하게 됩니다.

```tsx
export const prefetchData = async (queryClient: QueryClient) => {
  await queryClient.prefetchQuery(["data"], () => fetch());
};
```

이 함수가 실행된 후 다음 데이터를 조회할 때 미리 가져온 데이터를 활용하여 로딩화면 없이 빠르게 보여줄 수 있습니다.

## 결론

지금까지 React-Query를 사용한 여러 서버 상태 관리 기법을 소개했습니다. 유저가 어떻게 행동하는지 잘 파악하여 상황에 맞는 관리 전략을 이용한다면 사용자에게 더 큰 가치를 전달해줄 수 있을 것이라 생각합니다.

## 참고

[코드 예시](https://github.com/jinhokim98/technical-writing-example)
