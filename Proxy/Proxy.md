# Vì sao đoạn code triển khai này không bị revert với lỗi InvalidInitialization() mặc dù Initializable có một constructor và hàm initialize cũng kiểm tra biến initialized.

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

---

# Vì sao khi gọi delegatecall, không sử dụng storage của contract impl mà lại bị storage collision?

---

Đây là một câu hỏi rất hay, đi thẳng vào vấn đề cốt lõi và là một trong những rủi ro lớn nhất của các mẫu proxy không được thiết kế cẩn thận!

Bạn hỏi: "Tại sao khi gọi `delegatecall`, không sử dụng storage của contract `impl` mà lại bị storage collision?"

Câu trả lời ngắn gọn là: **Chính vì `delegatecall` KHÔNG sử dụng storage của contract `impl` mà thay vào đó sử dụng storage của contract `proxy`, nên mới xảy ra nguy cơ `storage collision` nếu thiết kế không đúng.**

Hãy cùng đi sâu hơn để làm rõ điều này:

### `delegatecall` và Shared Storage

Khi hợp đồng **A (Proxy)** gọi `delegatecall` đến hợp đồng **B (Implementation)**:

1.  Code của hợp đồng **B** được thực thi.
2.  Tuy nhiên, code của hợp đồng B được thực thi **trong ngữ cảnh của hợp đồng A**.
3.  Điều này có nghĩa là, khi code của B truy cập (đọc hoặc ghi) vào bất kỳ **biến trạng thái** nào, nó sẽ thao tác trên **storage của hợp đồng A (Proxy)**, chứ không phải storage của chính hợp đồng B.

Tức là, nếu hợp đồng `impl` của bạn có một biến `uint256 public value;` ở **slot storage 0**, và hợp đồng `proxy` của bạn cũng có một biến `address public admin;` ở **slot storage 0**, thì khi proxy gọi `delegatecall` đến `impl` và `impl` cố gắng thay đổi `value`, nó thực chất sẽ thay đổi `admin` của proxy! Đây chính là **`storage collision`**.

### Minh họa về `Storage Collision`

Hãy tưởng tượng storage của một hợp đồng như một mảng các ô nhớ (slots), mỗi ô 32 byte, được đánh số từ 0 trở đi.

**Kịch bản 1: Không có xung đột (Ideal Scenario)**

-   **Proxy Contract (`TimelockUpgradeableProxy`):**

    -   `_implementation` (ở slot `2**128`)
    -   `_admin` (ở slot `2**128 + 1`)
    -   ... (các biến của proxy ở các slot rất cao, không thể đoán trước)

-   **Implementation Contract (`AddressRegistry`):**
    -   `_upgradeOwner` (ở slot 0)
    -   `_pauseOwner` (ở slot 1)
    -   `_feeReceiver` (ở slot 2)
    -   ... (các biến của `AddressRegistry` ở các slot thấp)

Khi proxy gọi `delegatecall` đến `AddressRegistry` và `AddressRegistry` thay đổi `_upgradeOwner` của nó, nó sẽ ghi vào **slot 0 của proxy**. Nếu proxy không có biến nào ở slot 0, hoặc nếu nó là một biến không quan trọng (ví dụ như một biến padding), thì không có xung đột. Các biến của proxy được an toàn ở các slot rất cao.

**Kịch bản 2: Xung đột (Collision)**

-   **Proxy Contract:**

    -   `bool _initialized;` (ở slot 0)
    -   `address _implementation;` (ở slot 1)
    -   `address _admin;` (ở slot 2)

-   **Implementation Contract (`ERC20Token`):**
    -   `string public name;` (ở slot 0, nếu là biến đầu tiên)
    -   `string public symbol;` (ở slot 1)
    -   `uint256 public totalSupply;` (ở slot 2)

Nếu proxy gọi `delegatecall` đến `ERC20Token` và `ERC20Token` cố gắng khởi tạo `name` của nó, nó sẽ ghi vào **slot 0 của proxy**, nơi biến `_initialized` của proxy đang nằm. Điều này sẽ phá vỡ trạng thái của proxy, hoặc ngược lại, logic của `ERC20Token` sẽ không hoạt động đúng vì nó đang ghi đè lên các biến của proxy.

### Tại sao `layout at (2 ** 128)` lại quan trọng?

Chính vì nguy cơ `storage collision` này mà các mẫu proxy hiện đại (như ERC-1967 mà bạn thấy) sử dụng kỹ thuật đặt các biến trạng thái quan trọng của proxy vào các **slot bộ nhớ rất cao và ít có khả năng bị trùng lặp** (ví dụ: `2**128` hoặc `keccak256("erc1967.proxy.implementation") - 1`).

Mục đích là để tạo ra một "vùng an toàn" trong storage của proxy, nơi các biến của proxy có thể sống mà không sợ bị ghi đè bởi code của implementation (vì implementation thường chỉ sử dụng các slot thấp, bắt đầu từ 0).

---

Vậy nên, tóm lại, `storage collision` xảy ra **không phải vì `impl` sử dụng storage của chính nó**, mà chính xác là vì **`impl` KHÔNG sử dụng storage của chính nó mà lại thao tác trên storage của `proxy`**, và nếu cả proxy và `impl` đều định nghĩa các biến ở cùng một vị trí (slot) trong layout bộ nhớ của chúng, thì khi `impl` chạy, nó sẽ ghi đè lên biến của proxy (hoặc ngược lại).

---

# Như vậy, trong trường hợp thứ nhất. Ở các slot 0,1,2... của proxy, không có tên biến, nhưng vẫn có giá trị của biến ở đó đúng không?

Chính xác! Bạn đã nắm bắt được một điểm rất quan trọng về cách Solidity tổ chức **storage (bộ nhớ vĩnh viễn)** của hợp đồng.

Khi bạn nói: "Ở các slot 0, 1, 2... của proxy, không có tên biến, nhưng vẫn có giá trị của biến ở đó đúng không?"

Câu trả lời là **ĐÚNG TUYỆT ĐỐI!**

### Giải thích chi tiết

1.  **Storage là một mảng lớn các "slots"**:

    -   Trong EVM (Ethereum Virtual Machine), storage của mỗi hợp đồng thực chất là một mảng (array) rất lớn các ô nhớ 32-byte, được đánh số từ 0, 1, 2, ... cho đến $2^{256} - 1$. Mỗi ô này được gọi là một **slot**.
    -   Khi bạn khai báo một biến trạng thái trong Solidity, trình biên dịch sẽ gán cho nó một hoặc nhiều slot cụ thể.

2.  **Tên biến chỉ là "nhãn" cho trình biên dịch**:

    -   Trong code Solidity của bạn, bạn đặt tên biến (`owner`, `totalSupply`, v.v.). Các tên này giúp bạn (developer) dễ đọc và hiểu code.
    -   Tuy nhiên, khi code được biên dịch thành bytecode EVM, các tên biến này sẽ **biến mất**. Thay vào đó, mọi thao tác đọc/ghi sẽ chỉ tham chiếu đến các **số slot cụ thể**.

3.  **Proxy và Implementation chia sẻ "mảng slots"**:
    -   Khi một **Proxy** gọi `delegatecall` đến một **Implementation**, hợp đồng `Implementation` sẽ thực thi code của nó, nhưng khi nó đọc hoặc ghi vào một slot storage (ví dụ: slot 0, slot 1), nó sẽ thao tác trực tiếp trên **mảng slots của hợp đồng Proxy**.
    -   Hợp đồng Proxy không "biết" các tên biến của `Implementation`. Nó chỉ cung cấp không gian storage của nó.

### Ví dụ minh họa lại

Hãy xem lại kịch bản lý tưởng chúng ta đã nói:

-   **Proxy Contract (`TimelockUpgradeableProxy`):**

    -   Có thể có các biến của riêng nó ở các **slot rất cao** (ví dụ: địa chỉ implementation ở slot $2^{128}$).
    -   Giả sử **Proxy không khai báo bất kỳ biến nào ở các slot 0, 1, 2**... (chúng ta đang nói về "không có tên biến" ở đây).

-   **Implementation Contract (`AddressRegistry`):**
    -   `address public _upgradeOwner;` (trình biên dịch sẽ gán vào **slot 0** của `AddressRegistry` nếu nó là biến đầu tiên).
    -   `address public _pauseOwner;` (trình biên dịch sẽ gán vào **slot 1** của `AddressRegistry`).
    -   `address public _feeReceiver;` (trình biên dịch sẽ gán vào **slot 2** của `AddressRegistry`).

Khi người dùng gọi hàm qua Proxy, và Proxy dùng `delegatecall` tới `AddressRegistry`:

-   Nếu code của `AddressRegistry` ghi `_upgradeOwner = userAddress;`:
    -   Nó sẽ ghi `userAddress` vào **slot 0 của hợp đồng Proxy**.
    -   Dù Proxy không hề có tên biến nào ở slot 0, slot đó vẫn chứa giá trị `userAddress`.
-   Nếu code của `AddressRegistry` đọc `_pauseOwner;`:
    -   Nó sẽ đọc giá trị từ **slot 1 của hợp đồng Proxy**.
    -   Nếu chưa có gì được ghi vào slot 1 của Proxy, giá trị đọc được sẽ là 0 (hoặc giá trị mặc định của kiểu dữ liệu).

### Kết luận

Đúng vậy, các slot storage trên blockchain là các vị trí bộ nhớ vật lý. Chúng có thể chứa dữ liệu mà không nhất thiết phải tương ứng với một tên biến cụ thể trong code Solidity của hợp đồng đó. Khi `delegatecall` xảy ra, code của implementation chỉ biết về các số slot mà nó cần thao tác, và nó sẽ thao tác trên các slot đó trong không gian storage của hợp đồng proxy. Đây là lý do tại sao việc căn chỉnh slot (storage layout) cẩn thận giữa proxy và implementation là cực kỳ quan trọng để tránh `storage collision`.

---
