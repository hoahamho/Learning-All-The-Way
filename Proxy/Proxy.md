Đây là contract Initializable:

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity >=0.8.29;

import {InvalidInitialization} from "../interfaces/Errors.sol";

contract Initializable {
    bool private initialized;

    constructor() {
        initialized = true;
    }

    function initialize(bytes calldata data) external {
        if (initialized) revert InvalidInitialization();
        initialized = true;
        _initialize(data);
    }

    function _initialize(bytes calldata data) internal virtual {}
}

```

và `contract AddressRegistry is Initializable`

Đây là contract deploy:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.29;

import "forge-std/src/Script.sol";
import {AddressRegistry} from "../src/proxy/AddressRegistry.sol";
import {TimelockUpgradeableProxy} from "../src/proxy/TimelockUpgradeableProxy.sol";
import {Initializable} from "../src/proxy/Initializable.sol";

contract DeployAddressRegistry is Script {
    address constant UPGRADE_ADMIN = 0x02479BFC7Dce53A02e26fE7baea45a0852CB0909;
    address constant PAUSE_ADMIN = 0x02479BFC7Dce53A02e26fE7baea45a0852CB0909;
    address constant FEE_RECEIVER = 0x02479BFC7Dce53A02e26fE7baea45a0852CB0909;

    function run() public {
        vm.startBroadcast();
        address impl = address(new AddressRegistry());
        TimelockUpgradeableProxy proxy = new TimelockUpgradeableProxy(
            impl,
            abi.encodeWithSelector(
                Initializable.initialize.selector,
                abi.encode(UPGRADE_ADMIN, PAUSE_ADMIN, FEE_RECEIVER)
            )
        );

        console.log("AddressRegistry deployed at", address(proxy));
        vm.stopBroadcast();
    }
}

```

Vì sao không bị revert với lỗi `InvalidInitialization()`?

---

Đây là một câu hỏi rất hay, đi sâu vào trọng tâm của proxy pattern và cách hàm `initialize()` được thiết kế để hoạt động.

Bạn hỏi vì sao đoạn code triển khai này không bị revert với lỗi `InvalidInitialization()` mặc dù `Initializable` có một constructor và hàm `initialize` cũng kiểm tra biến `initialized`.

Lý do nằm ở cách **Proxy Pattern (cụ thể là UUPS)** hoạt động và sự khác biệt giữa việc gọi constructor trực tiếp so với gọi một hàm thông qua `delegatecall`.

---

### Phân tích vấn đề

Hãy xem xét lại contract `Initializable` và `AddressRegistry` (kế thừa `Initializable`):

```solidity
// contract Initializable
contract Initializable {
    bool private initialized; // Biến trạng thái của Initializable

    constructor() { // Constructor của Initializable
        initialized = true;
    }

    function initialize(bytes calldata data) external { // Hàm khởi tạo công khai
        if (initialized) revert InvalidInitialization(); // Kiểm tra initialized
        initialized = true;
        _initialize(data);
    }

    function _initialize(bytes calldata data) internal virtual {}
}
```

Và đoạn code triển khai:

```solidity
// contract DeployAddressRegistry
function run() public {
    vm.startBroadcast();
    address impl = address(new AddressRegistry()); // (1) Triển khai AddressRegistry (implementation)
    TimelockUpgradeableProxy proxy = new TimelockUpgradeableProxy( // (2) Triển khai Proxy
        impl,
        abi.encodeWithSelector(
            Initializable.initialize.selector,
            abi.encode(UPGRADE_ADMIN, PAUSE_ADMIN, FEE_RECEIVER)
        )
    );
    console.log("AddressRegistry deployed at", address(proxy));
    vm.stopBroadcast();
}
```

---

### Lý do không bị Revert `InvalidInitialization()`

Có hai sự kiện diễn ra ở đây, và mỗi sự kiện tương tác với biến `initialized` theo cách riêng của nó:

#### 1\. Constructor của `AddressRegistry` được gọi khi triển khai `impl` (implementation)

-   Dòng `address impl = address(new AddressRegistry());` sẽ triển khai một instance của `AddressRegistry` lên blockchain.
-   Khi `AddressRegistry` (hoặc `Initializable` mà nó kế thừa) được triển khai, **constructor của `Initializable` sẽ được gọi**.
-   Constructor này đặt `initialized = true;` **trong bộ nhớ vĩnh viễn (persistent storage) của chính hợp đồng `AddressRegistry` (implementation) đó**.
-   **Quan trọng:** `initialized` của hợp đồng `AddressRegistry` hiện là `true`. Nhưng đây là `initialized` CỦA IMPLEMENTATION. Biến này sẽ **không bao giờ được sử dụng** bởi người dùng cuối vì họ sẽ tương tác với proxy.

#### 2\. Hàm `initialize()` được gọi thông qua `proxy` (bằng `delegatecall`)

-   Dòng `TimelockUpgradeableProxy proxy = new TimelockUpgradeableProxy(...)` triển khai hợp đồng proxy.
-   Ngay sau khi proxy được triển khai, nó sẽ thực hiện một lệnh `delegatecall` nội bộ đến địa chỉ `impl` (là `AddressRegistry`) và gọi hàm `initialize()` với các tham số đã được mã hóa.
-   Khi `initialize()` được gọi qua `delegatecall` từ proxy, nó sẽ thực thi code của `AddressRegistry` **NHƯNG TRONG NGỮ CẢNH BỘ NHỚ VĨNH VIỄN CỦA HỢP ĐỒNG PROXY**.
-   Tại thời điểm này, biến `initialized` **trong bộ nhớ của hợp đồng proxy** (nơi mà tất cả các biến trạng thái thực sự được lưu trữ và truy cập bởi người dùng thông qua proxy) đang có giá trị mặc định ban đầu là `false`.
-   Vì `initialized` trong storage của proxy là `false`, điều kiện `if (initialized) revert InvalidInitialization();` **không được thỏa mãn**.
-   Sau đó, `initialized` trong storage của proxy được đặt thành `true;`, và `_initialize()` của `AddressRegistry` được gọi để hoàn tất quá trình thiết lập.

---

### Tóm lại:

Hợp đồng implementation (`AddressRegistry`) có một biến `initialized` trong storage của riêng nó (được constructor của nó set là `true`), nhưng biến này bị bỏ qua hoàn toàn trong môi trường proxy. Biến `initialized` **thực sự quan trọng** là biến nằm trong **storage của hợp đồng proxy**. Biến này ban đầu là `false`, cho phép hàm `initialize()` được gọi thành công lần đầu tiên qua proxy.

Đây chính là lý do tại sao các hợp đồng implementation trong proxy pattern thường có hàm `initialize()` thay vì `constructor` để thiết lập trạng thái ban đầu một cách an toàn và đúng đắn trong ngữ cảnh của proxy.
