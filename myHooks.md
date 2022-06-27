## lifeCycle

### 1. useOnMount

```ts
/**
 * 组件内只执行一次, 内部支持 async await
 * @param fn 要执行方法
 * @param destoryCallBack 组件销毁执行的回调
 */
export function useOnMount(fn: () => void, destoryCallBack?: () => void) {
    // eslint-disable-next-line consistent-return
    useEffect(() => {
        fn();
        if (destoryCallBack) {
            return () => {
                destoryCallBack();
            };
        }
    // eslint-disable-next-line react-hooks/exhaustive-deps
    }, []);
}
```

### 2. useOnUpdate

```ts	
/**
 *  第一次不执行，组件更新执行
 * @param fn 要执行方法
 * @param dep 所执行的方法依赖哪些状态的变化
 */
export function useOnUpdate(fn: () => void, dep?: any[]) {
    const ref = useRef({fn, mounted: false});
    ref.current.fn = fn;

    useEffect(() => {
        // 首次渲染不执行
        // eslint-disable-next-line no-negated-condition
        if (!ref.current.mounted) {
            ref.current.mounted = true;
        } else {
            ref.current.fn();
        }
    // eslint-disable-next-line react-hooks/exhaustive-deps
    }, dep);
}
```

### 3. useForceUpdate

```ts
// 强制更新组件，不提倡，除非万不得已
export function useForceUpdate() {
    const [, setValue] = useState(0);
    return useCallback(() => {
        // 递增state值，强制React进行重新渲染
        setValue(val => (val + 1) % (Number.MAX_SAFE_INTEGER - 1));
    }, []);
}

```

### 4. useHttp

```ts
/**
 *
 * @param url 请求URL
 * @param reqParams 请求参数
 * @param isImmediately 是否立即触发一次请求
 */
export function useHttp<Req, Res, ResOrigin = Res>(
    httpConfig: {
        methods: NetMethods;
        url: string;
        params?: Req;
        // 数据转换
        transform?: (p: ResOrigin) => Res;
        succCallBack?: (p: Res) => void;
        errorCallBack?: (p: any) => void;
    },
    isImmediately?: boolean
): [Res, (params: Req) => Promise<Res>] {
    const [data, setData] = useState<Res>(null);
    const http = (params: Req = httpConfig.params): Promise<Res> => {
        return new Promise((resolve, reject) => {
            net[httpConfig.methods]<Req, ResponseObj<Res>>(httpConfig.url, params).then(res => {
                const resRes = httpConfig.transform ? httpConfig.transform((res.result as any)) : res.result;
                setData(resRes);
                httpConfig.succCallBack && httpConfig.succCallBack(resRes);
                resolve(resRes);
            }).then(err => {
                httpConfig.errorCallBack && httpConfig.errorCallBack(err);
                reject(err);
            });
        });
    };
    useOnMount(() => {
        if (isImmediately) {
            http(httpConfig.params);
        }
    });
    return [
        data,
        http,
    ];
}

```

### 5. useStateRef

```ts
// 对state的扩展
// 组件渲染更新用 state 对于立即要用的变量直接使用ref
// 这符合我们对数据的直觉，不用考虑闭包和capture value特性

export function useStateRef<T>(initialState: T | (() => T)): [
    T,
    Dispatch<SetStateAction<T>>,
    React.MutableRefObject<T>
] {
    const refContainer = useRef<T>();

    const [state, setState] = useState<T>(() => {
        // 初始化保留state的可以传入函数的特性
        const value = isFunction(initialState) ? (initialState as any)() : initialState;
        refContainer.current = value;
        return value;
    });

    // 把setState的原始用法也保留
    const setValue = useCallback(value => {
        if (isFunction(value)) {
            setState((prevState: T) => {
                const newState = value(prevState);
                refContainer.current = newState;
                return newState;
            });
        } else {
            setState(value);
            refContainer.current = value;
        }
    }, []);

    return [state, setValue, refContainer];
}

```