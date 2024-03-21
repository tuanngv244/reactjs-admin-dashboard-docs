# Authentication API flow

Xây dựng luồng Anthentication với Redux toolkit và PrivateRoute xác thực user có quyền.

## 1. Redux auth module

- Dựng redux cho phần auth
- Xử lý function `_onLogin` để đăng nhập
- Xử lý function `handleGetProfile` để lấy thông tin user

#### Dựng redux cho phần auth

- Trong folder store/modules/auth tạo file index.ts
- Define type `AuthState` và khai báo `initialState`.
- Tạo slice bằng `createSlice` cho auth.
- Tạo function `reset` khi muốn reset state về `initialState` của auth module.
- Xuất `authActions` và `authReducer` để bên ngoài có thể sử dụng.

```tsx
type AuthState = {
  profile?: IUser;
  loading: {
    getProfile: boolean;
  };
};

const initialState: AuthState = {
  loading: {
    getProfile: false,
  },
};

const { actions, reducer: authReducer } = createSlice({
  name: "auth",
  initialState,
  reducers: {
    reset() {
      return initialState;
    },
  },
  extraReducers: (builder) => {},
});

const authActions = { ...actions };
export { authActions, authReducer };
```

#### Xử lý hàm `_onLogin` để đăng nhập.

- Tạo hàm `createAsyncThunk` để handle action side effects.
- Function `handleLogin` nhận vào payload là data có type là `ILoginFormData`.
- Function `handleLogin` cũng nhận thêm 2 tham số là function `onSuccess` và `onFailed`.
- Nếu gọi `login` thành công lấy `token` và `id` lưu vào Cookie với `tokenMethod.set({...})`.
- Sử dụng `thunkApi.dispatch(handleGetProfile(id))` để lấy thông tin user sau đăng nhập.
- Gọi function `onSuccess` nếu `login` thành công và function `onFailed` nếu gọi thất bại.

```tsx
...

export const handleLogin = createAsyncThunk(
  "auth/login",
  async (
    args: {
      payload: ILoginFormData;
      onSuccess?: VoidFunction;
      onFailed?: VoidFunction;
    },
    thunkApi
  ) => {
    const { payload, onSuccess, onFailed } = args;
    try {
      const res = await authService.login(payload);
      const { id, token } = res?.data || {};
      thunkApi.dispatch(handleGetProfile(id));
      tokenMethod.set({
        id: id,
        accessToken: token,
        refreshToken: token,
        countRefreshToken: 0,
      });
      onSuccess?.();
      return res?.data;
    } catch (error) {
      onFailed?.();
      return thunkApi.rejectWithValue(error);
    }
  }
);

...

const authActions = { ...actions, };
export { authActions, authReducer };
```

#### Xử lý function `handleGetProfile` để lấy thông tin user

- Tạo hàm `createAsyncThunk` để handle action side effects.
- Function `handleGetProfile` nhận vào id để gọi api `getProfile`.

```tsx
...

export const handleGetProfile = createAsyncThunk(
  "auth/getProfile",
  async (id: string, thunkApi) => {
    try {
      const profileRes = await authService.getProfile(id);
      return profileRes;
    } catch (error) {
      return thunkApi.rejectWithValue(error);
    }
  }
);

...

const authActions = { ...actions, };
export { authActions, authReducer };
```

- Dựng `builder` để bắt data và các trạng thái của function `handleGetProfile`.
- Nếu `fulfilled` lưu thông tin profile vào store.

```tsx
...

const { actions, reducer: authReducer } = createSlice({
  name: "auth",
  initialState,
  reducers: {
    reset() {
      return initialState;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(handleGetProfile.fulfilled, (state, action) => {
        state.profile = action.payload;
        state.loading.getProfile = false;
      })
      .addCase(handleGetProfile.pending, (state) => {
        state.loading.getProfile = true;
      })
      .addCase(handleGetProfile.rejected, (state) => {
        state.loading.getProfile = false;
      });
  },
});

...
```

## 2. Login form page

- Dựng `Authentication` page và config route
- Xử lý `Login` component

#### Dựng Authentication page và config route

- Tạo page `Authentication` với cấu trúc `pages/Authentication/index.tsx`.
- Config route với path `/login` cho page `Authentication` trên.
- Logic kiểm tra sự tồn tại token:
  - Nếu có token thì sẽ `Navigate` về trang `Dashboard`.
  - Nếu không có token thì sẽ render component `Login`.

```tsx
/// Authentication page
const Authentication: React.FC = () => {
  return !tokenMethod.get() ? <Login /> : <Navigate to={"/"} replace />;
};

export default Authentication;
```

```tsx
/// Config route tại configs/routes.tsx
...

const Authentication = withLoader(lazy(() => import('../pages/Authentication')));

...

  {
    path: Paths.ROOT,
    element: <MainLayout />,
    children: [
      {
        path: Paths.AUTHENTICATION,
        element: <Authentication />,
      },
      ...
    ]
  }
```

#### Xử lý Login component

- Dùng `useForm` trong `react-hook-form` để xử lý form login.
- Tạo type cho form là `ILoginFormData` định nghĩa các trường dữ liệu gửi đi.
- Dựng form login UI.

```tsx
/// Form login UI
...
const Login = () => {

const { register, handleSubmit } = useForm<ILoginFormData>({});

const _onLogin = async (data: ILoginFormData) =>{}

<Box
  sx={{
  width: "100%",
  height: "100%",
  display: "flex",
  alignItems: "center",
  justifyContent: "center",
  }}
  >
  <Box
    sx={{
    padding: "32px",
    maxWidth: "400px",
    }}
    component="form"
    onSubmit={handleSubmit(_onLogin)}
    >
    <Box sx={{ mb: 5 }}>
      <Typography variant="h3">{t("login")}</Typography>
    </Box>
    <Input placeholder="Email" fullWidth {...register("email")} />
    <Input
      type="password"
      placeholder="Mật khẩu"
      fullWidth
      {...register("password")}
      />
    <Button fullWidth type="submit">
      Đăng nhập
    </Button>
  </Box>
</Box>
};
export default Login;
```

- Xử lý function `_onLogin` gọi `dispatch` action `authActions.handleLogin` để gọi API login.
- Xử lý trạng thái login:
  - Nếu thành công function `onSuccess` thực thi, dùng `navigate` với path `Paths.ROOT` để vào trang `Dashboard`.
  - Nếu thất bại bắn một message error thông báo cho người dùng.

```tsx
const Login = () => {
  const { register, handleSubmit } = useForm<ILoginFormData>({});
  const dispatch = useDispatch<AppDispatch>();
  const navigate = useNavigate();
  const { t } = useTranslation();

  // Xử lý function _onLogin
  const _onLogin = async (data: ILoginFormData) =>
    await dispatch(
      authActions.handleLogin({
        payload: data,
        onSuccess: () => navigate(Paths.ROOT),
        onFailed: () => alert("Login failed!"),
      })
    );

  return (
    <Box
      sx={{
        width: "100%",
        height: "100%",
        display: "flex",
        alignItems: "center",
        justifyContent: "center",
      }}
    >
      <Box
        sx={{
          padding: "32px",
          maxWidth: "400px",
        }}
        component="form"
        onSubmit={handleSubmit(_onLogin)}
      >
        <Box sx={{ mb: 5 }}>
          <Typography variant="h3">{t("login")}</Typography>
        </Box>
        <Input placeholder="Email" fullWidth {...register("email")} />
        <Input
          type="password"
          placeholder="Mật khẩu"
          fullWidth
          {...register("password")}
        />
        <Button fullWidth type="submit">
          Đăng nhập
        </Button>
      </Box>
    </Box>
  );
};

export default Login;
```

## 3. Kiểm tra route với PrivateRoute

- Dựng `PrivateRoute` với cấu trúc folder components/PrivateRoute/index.tsx
- PrivateRoute nhận 2 props `redirectTo`,`element`:
  - `redirectTo` dùng để định nghĩa route redirect khi user không có quyền truy cập.
  - `element` dùng để render route đó khi user có quyền truy cập.
- Xử lý kiểm tra sự tồn tại của token:
  - Nếu có token thì sẽ return `element`.
  - Nếu không có token thì sẽ `Naviagte` đến component có path là `redirectTo`.

```tsx
interface PrivateRouteProps {
  redirectTo: string;
  element: React.ReactNode;
}

const PrivateRoute: React.FC<PrivateRouteProps> = ({
  redirectTo = "/authen",
  element,
}) => {
  return tokenMethod.get() ? element : <Navigate to={redirectTo} replace />;
};

export default PrivateRoute;
```

- Khai báo PrivateRoute với path `Paths.ROOT` làm cổng chặn mọi route
  con đều phải đi qua.
- Route con được khai báo trong `children`.

```tsx
...
 {
    path: Paths.ROOT,
    element: <MainLayout />,
    children: [
      {
        path: Paths.AUTHENTICATION,
        element: <Authentication />,
      },
      {
        path: Paths.ROOT,
        element: <PrivateRoute redirectTo={Paths.AUTHENTICATION} element={<MenuLayout />} />,
        children:[
          ...
        ]
      }
      ...
    ]
 }
```
