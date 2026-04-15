# IL2CPP Offset Finder for Unity Android Games

Thư viện này giúp tìm địa chỉ method và offset field trong game Unity sử dụng IL2CPP trên Android mà không cần dùng Il2CppDumper.

## Tính năng

- Tự động tìm đường dẫn `libil2cpp.so` và `global-metadata.dat`
- Giải mã metadata bị obfuscate (nếu có)
- Parse ELF sections của `libil2cpp.so` đang được load
- Tìm cấu trúc `Il2CppCodeRegistration` và `Il2CppMetadataRegistration`
- Lấy con trỏ method từ tên (image, class, method, số tham số)
- Lấy offset field (tự động xử lý value type)

## Yêu cầu

- Android NDK, C++11
- Quyền đọc file trong `/proc/self/maps` và data directory của app

## Cách dùng

### 1. Thêm file `.h` vào project

Copy toàn bộ nội dung file `UnitySDK.h` (đã cung cấp) vào project của bạn.

### 2. Khởi tạo cache

Gọi `Unity::EnsureCache()` một lần duy nhất, thường ở `JNI_OnLoad`:

```cpp
#include "unity_offset_finder.h"

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    if (!Unity::EnsureCache()) {
        __android_log_print(ANDROID_LOG_ERROR, "Mod", "Init failed");
        return JNI_ERR;
    }
    // ... tiếp tục
    return JNI_VERSION_1_6;
}

// Lấy base address của libil2cpp.so (có thể dùng dlopen)
void* libil2cpp = dlopen("libil2cpp.so", RTLD_LAZY);
uintptr_t il2cpp_base = (uintptr_t)libil2cpp;

// Lấy địa chỉ hàm Jump trong class Player (Assembly-CSharp, không namespace)
void* jump_func = Unity::GetMethodAddress(il2cpp_base,
                                          "Assembly-CSharp",
                                          "",           // namespace (rỗng nếu không có)
                                          "Player",
                                          "Jump",
                                          0);           // số tham số
if (jump_func) {
    // Ép kiểu và gọi
    ((void(*)(void*))jump_func)(playerObject);
}

uintptr_t health_offset = Unity::GetFieldOffset("Assembly-CSharp",
                                                "",
                                                "Player",
                                                "health");
if (health_offset) {
    int* health_ptr = (int*)((char*)playerObject + health_offset);
    *health_ptr = 100;
}

namespace Unity {
    // Khởi tạo cache (gọi 1 lần)
    bool EnsureCache();

    // Trả về RVA của method (0 nếu không tìm thấy)
    uint64_t FindMethodOffset(const char* image_name,
                              const char* type_namespace,
                              const char* type_name,
                              const char* method_name,
                              int argsCount);

    // Trả về con trỏ hàm (thêm base address)
    inline void* GetMethodAddress(uintptr_t il2cpp_base,
                                  const char* image_name,
                                  const char* type_namespace,
                                  const char* type_name,
                                  const char* method_name,
                                  int argsCount);

    // Trả về offset field (byte)
    int FindFieldOffset(const char* image_name,
                        const char* type_namespace,
                        const char* type_name,
                        const char* field_name);

    // Inline wrapper
    inline uintptr_t GetFieldOffset(const char* image_name,
                                    const char* type_namespace,
                                    const char* type_name,
                                    const char* field_name);
}
