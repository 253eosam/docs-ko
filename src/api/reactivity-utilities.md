# 반응형 API: 유틸리티 {#reactivity-api-utilities}

## isRef() {#isref}

값이 ref 객체인지 확인합니다.

- **타입**

  ```ts
  function isRef<T>(r: Ref<T> | unknown): r is Ref<T>
  ```

  반환 타입은 [type predicate](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)이므로,
  `isRef`를 타입 가드로 사용할 수 있습니다.

  ```ts
  let foo: unknown
  if (isRef(foo)) {
    // foo의 타입은 Ref<unknown>으로 한정됨
    foo.value
  }
  ```

## unref() {#unref}

인자가 ref이면 내부 값을 반환하고, 그렇지 않으면 인자 자체를 반환합니다.
이것은 `val = isRef(val) ? val.value : val`과 같습니다.

- **타입**

  ```ts
  function unref<T>(ref: T | Ref<T>): T
  ```

- **예제**

  ```ts
  function useFoo(x: number | Ref<number>) {
    const unwrapped = unref(x)
    // unwrapped는 이제 확실히 숫자 입니다
  }
  ```

## toRef() {#toref}

값 / ref / getter 들을 정규화하는 데 사용할 수 있습니다 (3.3+).

또는 반응형 객체의 속성에 대한 ref를 만드는 데 사용할 수도 있습니다.
생성된 ref는 소스 속성과 동기화됩니다.
소스 속성을 변경하면 ref가 업데이트되고 그 반대의 경우도 마찬가지입니다.

- **타입**

  ```ts
  // 정규화 시그니처 (3.3+)
  function toRef<T>(
    value: T
  ): T extends () => infer R
    ? Readonly<Ref<R>>
    : T extends Ref
    ? T
    : Ref<UnwrapRef<T>>

  // 객체 속성 시그니처
  function toRef<T extends object, K extends keyof T>(
    object: T,
    key: K,
    defaultValue?: T[K]
  ): ToRef<T[K]>

  type ToRef<T> = T extends Ref ? T : Ref<T>
  ```

- **예제**

  정규화 시그니처 (3.3+):

  ```js
  //  기존 참조를 있는 그대로 반환함
  toRef(existingRef)

  // .value로 접근했을 때 getter를 호출하는 읽기 전용 ref를 만듦
  toRef(() => props.foo)

  // 함수가 아닌 값으로부터 일반적인 ref들을 만듦
  // ref(1) 과 동일
  toRef(1)
  ```

  객체 속성 시그니처:

  ```js
  const state = reactive({
    foo: 1,
    bar: 2
  })

  // 원래 속성과 동기화되는 양방향 ref
  const fooRef = toRef(state, 'foo')

  // ref를 변경하면 원본도 업데이트 됨
  fooRef.value++
  console.log(state.foo) // 2

  // 원본을 변경하면 ref도 업데이트 됨
  state.foo++
  console.log(fooRef.value) // 3
  ```

  이것은 다음과 다름에 주의해야 합니다:

  ```js
  const fooRef = ref(state.foo)
  ```

  위의 ref는 `state.foo`와 **동기화되지 않습니다**.
  `ref()`가 일반 숫자 값을 수신하기 때문입니다.

  `toRef()`는 컴포저블 함수에 prop을 ref로 전달하려는 경우에 유용합니다:

  ```vue
  <script setup>
  import { toRef } from 'vue'
  
  const props = defineProps(/* ... */)

  // `props.foo`를 ref로 변환한 다음 컴포저블 함수에 전달
  useSomeFeature(toRef(props, 'foo'))
  
  // getter 문법 - 3.3 이상 버전에서 권장됨
  useSomeFeature(toRef(() => props.foo))
  </script>
  ```

  `toRef`가 컴포넌트 props와 함께 사용되면,
  props 변경에 대한 일반적인 제한 사항이 계속 적용됩니다.
  ref에 새 값을 할당하려는 시도는 prop을 직접 수정하려는 것과 동일하며 허용되지 않습니다.
  이런 경우에는 [`computed()`](./reactivity-core#computed)에 `get`과 `set`을 선언하여 사용하는 것으로 구현할 수 있습니다.
  자세한 내용은 [컴포넌트를 `v-model`과 함께 사용하기](/guide/components/v-model) 가이드 참고.

  객체 속성 시그니처를 사용할 때, `toRef()`는 원본 속성이 현재 존재하지 않더라도 사용 가능한 ref를 반환합니다. 이를 통해 [`toRefs`](#torefs)에서 감지되지 않는 선택적 속성들을 처리할 수 있게 됩니다.

## toValue() {#tovalue}

-  3.3+ 버전에서만 지원됩니다.


값, ref, 그리고 getter를 일반 값으로 변환(normalize)합니다. 이는 [unref()](#unref)와 유사하지만, getter도 변환한다는 점이 다릅니다. 만약 인수로 getter가 전달되면, 해당 getter가 실행되고 반환된 값이 반환됩니다.

이 기능은  [Composables](/guide/reusability/composables)에서 인수를 값, ref, 또는 getter 중 어떤 형태로든 받을 수 있도록 정규화할 때 사용할 수 있습니다.

- **타입**

  ```ts
  function toValue<T>(source: T | Ref<T> | (() => T)): T
  ```

- **예제**

  ```js
  toValue(1) //       --> 1
  toValue(ref(1)) //  --> 1
  toValue(() => 1) // --> 1
  ```

  컴포저블에서 인자를 정규화 하기:

  ```ts
  import type { MaybeRefOrGetter } from 'vue'

  function useFeature(id: MaybeRefOrGetter<number>) {
    watch(() => toValue(id), id => {
      //  id  변경에 대응
    })
  }

  // 이 컴포저블은 다음의 형식을 지원함
  useFeature(1)
  useFeature(ref(1))
  useFeature(() => 1)
  ```

## toRefs() {#torefs}

반응형 객체를 일반 객체로 변환하고,
변환된 일반 객체의 각 속성은 원본 객체(반응형 객체)의 속성이 ref된 것 입니다.
각 개별 ref는 [`toRef()`](#toref)를 사용하여 생성됩니다.

- **타입**

  ```ts
  function toRefs<T extends object>(
    object: T
  ): {
    [K in keyof T]: ToRef<T[K]>
  }

  type ToRef = T extends Ref ? T : Ref<T>
  ```

- **예제**

  ```js
  const state = reactive({
    foo: 1,
    bar: 2
  })

  const stateAsRefs = toRefs(state)
  /*
  stateAsRefs의 타입: {
    foo: Ref<number>,
    bar: Ref<number>
  }
  */

  // 원본 속성이 ref와 "연결됨"
  state.foo++
  console.log(stateAsRefs.foo.value) // 2

  stateAsRefs.foo.value++
  console.log(state.foo) // 3
  ```

  `toRefs`는 컴포저블 함수에서 반응형 객체를 반환하면,
  이것을 사용하는 컴포넌트가 반응형을 잃지 않고 분해 할당 및 확장 할 수 있어 유용합니다.

  ```js
  function useFeatureX() {
    const state = reactive({
      foo: 1,
      bar: 2
    })

    // ...state를 사용하여 작동하는 로직

    // 반환할 때 refs로 변환
    return toRefs(state)
  }

  // 반응형을 잃지 않고 분해 할당 가능
  const { foo, bar } = useFeatureX()
  ```

  `toRefs`는 호출 시 소스 객체에서 열거 가능한 속성만 참조로 생성합니다.
  아직 존재하지 않을 수 있는 속성에 대한 참조를 생성하려면 [`toRef`](#toref)를 사용해야 합니다.

## isProxy() {#isproxy}

객체가 [`reactive()`](./reactivity-core#reactive), [`readonly()`](./reactivity-core#readonly), [`shallowReactive()`](./reactivity-advanced#shallowreactive) 또는 [`shallowReadonly()`](./reactivity-advanced#shallowreadonly)에 의해 생성된 프록시인지 확인합니다.

- **타입**

  ```ts
  function isProxy(value: any): boolean
  ```

## isReactive() {#isreactive}

객체가 [`reactive()`](./reactivity-core#reactive) 또는 [`shallowReactive()`](./reactivity-advanced#shallowreactive)에 의해 생성된 프록시인지 확인합니다.

- **타입**

  ```ts
  function isReactive(value: unknown): boolean
  ```

## isReadonly() {#isreadonly}

전달된 값이 읽기 전용 객체인지 확인합니다. 읽기 전용 객체의 속성은 변경할 수 있지만 전달된 객체를 통해 직접 할당할 수는 없습니다.

[`readonly()`](./reactivity-core#readonly) 및 [`shallowReadonly()`](./reactivity-advanced#shallowreadonly)로 생성된 프록시는 모두 읽기 전용으로 간주되며, `set` 함수가 없는 [`computed()`](./reactivity-core#computed) 참조와 마찬가지로 마찬가지입니다.


- **타입**

  ```ts
  function isReadonly(value: unknown): boolean
  ```
