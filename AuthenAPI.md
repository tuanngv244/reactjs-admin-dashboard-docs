# Authentication API flow

## AuthContext

- state `profile` dùng để lưu trữ thông tin profile của account, truyền qua các component trong context sử dung5
- `handleLogin`:
  - Gọi API login với payload tương ứng (xem Swagger)
  - Gọi api thành công
    - Lưu JWT token vào localStorage hoặc cookies
    - Gọi handleGetProfile để lấy thông tin profile
    - Thông báo thành công và đóng modal
  - Gọi api thất bại
    - Thông báo thất bại
  - Kết thúc gọi API => gọi callback được truyền vào. Mục đích clear loading,...
- `handleRegister`:
  - Gọi API register với payload tương ứng (xem Swagger)
  - Gọi api thành công
    - Kiểm tra data trả về có id của accoung không
    - Gọi handleLogin đăng nhập và lấy thông tin profile
  - Gọi api thất bại
    - Thông báo thất bại
  - Kết thúc gọi API => gọi callback được truyền vào. Mục đích clear loading,...
- `handleLogout`: clear token ở localStorage hoặc cookies và clear profile data
- `handleGetProfile`:

  - Gọi API getProfile với kèm theo token (xem `authService.getProfile`)
  - Gọi thành công: thông báo thành công và cập nhật state profile
  - Gọi thất bại: thông báo thất bại và gọi `handleLogout`

- `handleLogin`: gọi API login với payload tương ứng (xem Swagger)
- `useEffect`: gọi lần đầu khi khởi chạy context để check token => gọi `handleGetProfile`
- `handleShowModal`: bổ sung điều kiện, nếu có token sẽ KHÔNG mở modal và ngược lại

```jsx
import { authService } from "@/services/authService";
import tokenMethod from "@/utils/token";
import { message } from "antd";
import { createContext, useContext, useEffect, useState } from "react";

const AuthContext = createContext({});

const AuthContextProvider = ({ children }) => {
  const [showedModal, setShowedModal] = useState("");
  const [profile, setProfile] = useState();

  const handleShowModal = (modalType) => {
    if (!!!tokenMethod.get()) {
      setShowedModal(modalType || "");
    }
  };

  const handleCloseModal = (e) => {
    e?.stopPropagation();
    setShowedModal("");
  };

  useEffect(() => {
    if (tokenMethod.get()) {
      // call api get profile
      handleGetProfile();
    }
  }, []);

  const handleLogin = async (loginData, callback) => {
    // call API
    try {
      const res = await authService.login(loginData);
      const { token: accessToken, refreshToken } = res?.data?.data || {};

      // Lưu vào local storage
      tokenMethod.set({
        accessToken,
        refreshToken,
      });

      if (!!tokenMethod) {
        // Lấy thông tin profile
        handleGetProfile();
        // Thông báo
        message.success("Đăng nhập thành công");

        // Đóng modal
        handleCloseModal();
      }
    } catch (error) {
      console.log("error", error);
      message.error("Đăng nhập thất bại");
    } finally {
      callback?.();
    }
  };

  const handleRegister = async (registerData, callback) => {
    // call API
    try {
      const { name, email, password } = registerData;
      const payload = {
        firstName: name,
        lastName: "",
        email,
        password,
      };
      const res = await authService.register(payload);
      if (res?.data?.data?.id) {
        message.success("Đăng ký thành công");
        handleLogin({
          email,
          password,
        });
      }
    } catch (error) {
      console.log("error", error);
      if (error?.response?.status === 403) {
        message.error("Email đăng ký đã tồn tại");
      } else {
        message.error("Đăng ký thất bại");
      }
    } finally {
      callback?.();
    }
  };

  const handleLogout = () => {
    tokenMethod.remove();
    setProfile(undefined);
  };

  const handleGetProfile = async () => {
    try {
      const profileRes = await authService.getProfile();
      if (profileRes?.data?.data) {
        setProfile(profileRes.data.data);
      }
    } catch (error) {
      console.log("error", error);
      handleLogout();
    }
  };

  return (
    <AuthContext.Provider
      value={{
        showedModal,
        profile,
        handleShowModal,
        handleCloseModal,
        handleLogin,
        handleLogout,
        handleRegister,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export default AuthContextProvider;

export const useAuthContext = () => useContext(AuthContext);
```

## authService.getProfile

- Đối với các API yêu cầu xác thực `(authenticated)` quyền truy cập, chúng ta cần điều chỉnh `request header` để gửi kèm `accessToken`

```jsx
import axiosInstance from "@/utils/axiosInstance";
import tokenMethod from "@/utils/token";

export const authService = {
  //  ...
  getProfile() {
    return axiosInstance.get(`/customer/profiles`, {
      headers: {
        Authorization: `Bearer ${tokenMethod.get()?.accessToken}`,
      },
    });
  },
  // ...
};
```

## token utils

- tạo helper functions `localToken` để tiện dùng khi xử lý token với localStorage
- tạo helper functions `cookieToken` để tiện dùng khi xử lý token với cookie. Lưu ý, cần cài đặt lib `js-cookie` để xử lý cookie tốt hơn

```jsx
// constants/storage.js
export const STORAGE = {
  token: "token",
};
```

```jsx
// utils/token.js
import { STORAGE } from "@/constants/storage";
import Cookies from "js-cookie";

export const localToken = {
  get: () => JSON.parse(localStorage.getItem(STORAGE.token)),
  set: (token) => localStorage.setItem(STORAGE.token, JSON.stringify(token)),
  remove: () => localStorage.removeItem(STORAGE.token),
};

export const cookieToken = {
  get: () =>
    JSON.parse(
      Cookies.get(STORAGE.token) !== undefined
        ? Cookies.get(STORAGE.token)
        : null
    ),
  set: (token) => Cookies.set(STORAGE.token, JSON.stringify(token)),
  remove: () => Cookies.remove(STORAGE.token),
};

const tokenMethod = {
  get: () => {
    // return localToken.get()
    return cookieToken.get();
  },
  set: (token) => {
    console.log("token", token);
    // localToken.set(token)
    cookieToken.set(token);
  },
  remove: () => {
    // localToken.remove();
    cookieToken.remove();
  },
};

export default tokenMethod;
```

## LoginForm

- Sau khi `_onSubmit` và validate thành công => gọi `handleLogin` được lấy từ AuthContext, truyền `form data` và `callback` function
- `callback` function với mục đích clear loading sau khi hoàn thành `handleLogin`

```jsx
const LoginForm = () => {
  const { handleLogin } = useAuthContext();
  const [loading, setLoading] = useState(false);

  const _onSubmit = async (e) => {
    e.preventDefault();
    // validation
    const errObj = {};
    // ...
    //end validation
    if (Object.keys(errObj)?.length > 0) {
      console.log("Submit error", errObj);
    } else {
      console.log("Submit success", form);
      setLoading(true);
      handleLogin?.({ ...form }, () => {
        setTimeout(() => {
          setLoading(false);
        }, 300);
      });
    }
  };
  // ...
};
```

## RegisterForm

- Sau khi `_onSubmit` và validate thành công => gọi `handleRegister` được lấy từ AuthContext, truyền `form data` và `callback` function
- `callback` function với mục đích clear loading sau khi hoàn thành `handleRegister`

```jsx
const RegisterForm = () => {
  const { handleRegister } = useAuthContext();
  const [loading, setLoading] = useState(false);

  const _onSubmit = (e) => {
    e.preventDefault();
    // validation
    const errObj = {};
    // ...
    //end validation
    if (Object.keys(errObj)?.length > 0) {
      console.log("Submit error", errObj);
    } else {
      setLoading(true);
      console.log("Submit success", form);
      handleRegister({ ...form }, () => {
        setTimeout(() => {
          setLoading(false);
        }, 300);
      });
    }
  };
  //   ...
};
```
