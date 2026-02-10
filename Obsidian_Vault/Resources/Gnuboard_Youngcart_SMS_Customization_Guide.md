### **그누보드 및 영카트 아이코드 문자 발송 기능 자체 API 연동 가이드 (개념적 접근)**

본 가이드는 `web_search` 도구 사용 불가로 인해 그누보드 및 영카트의 일반적인 구조와 외부 API 연동 패턴에 대한 지식을 기반으로 작성되었습니다. 실제 시스템에 적용 시 코드 및 파일 경로가 다를 수 있으므로, 반드시 백업 후 신중하게 작업하시기 바랍니다.

---

#### **1. 현재 아이코드 연동 방식 분석 (추정)**

그누보드 및 영카트는 외부 문자 발송 서비스를 연동할 때 일반적으로 다음과 같은 패턴을 따릅니다:

*   **전용 SMS 라이브러리/클래스 파일:** `lib/sms.lib.php` 또는 `extend/sms_extend.php` 등 SMS 관련 핵심 로직을 처리하는 파일이 존재합니다. 이 파일 내에서 아이코드(또는 다른 외부 SMS 서비스)의 API 연 호출 코드가 포함되어 있을 가능성이 높습니다.
*   **설정 파일:** `config.php`, `_common.php` 또는 별도의 SMS 설정 파일(`data/sms_conf.php` 등)에서 아이코드 연동에 필요한 API 키, 아이디, 비밀번호 등의 정보가 정의되어 있거나, 데이터베이스에서 불러오도록 설정되어 있습니다.
*   **문자 발송 트리거:** 회원가입, 주문 완료, 관리자 문자 발송 등 특정 이벤트 발생 시 위 라이브러리/클래스의 함수를 호출하여 문자를 발송합니다. 예: `register_form_update.php`, `shop/orderformupdate.php`, `adm/sms_form_update.php`.

현재 아이코드 연동 방식은 주로 이 핵심 라이브러리 파일 내에서 아이코드의 PHP SDK를 사용하거나, 직접 HTTP(POST/GET) 요청을 통해 아이코드 API 엔드포인트에 데이터를 전송하는 방식으로 구현되어 있을 것입니다.

#### **2. 자체 API 연동을 위한 코드 수정 지점 및 방법**

자체 API 서비스로 연동하기 위해서는 기존 아이코드 API 호출 로직을 찾아 자체 API 호출 로직으로 대체해야 합니다.

**2.1. 자체 API 서비스 인터페이스 정의**

먼저 자체 API 서비스가 어떤 엔드포인트를 통해 어떤 형식으로 데이터를 받아 문자를 발송할 것인지 설계해야 합니다. (예: `POST /api/sms/send`, `{"to": "010-1234-5678", "message": "안녕하세요.", "from": "02-123-4567"}`)

**2.2. 핵심 수정 지점 파악**

가장 중요한 수정 지점은 SMS 발송이 이루어지는 **핵심 함수 또는 클래스**입니다.

*   **`lib/sms.lib.php` (또는 유사 파일):**
    *   이 파일 내에서 아이코드 관련 함수(예: `icode_send_sms()`, `send_sms_icode()`)를 찾습니다.
    *   이 함수 내부의 아이코드 API 호출 부분을 자체 API 호출 로직으로 변경합니다.
    *   **대안:** 기존 아이코드 함수는 유지하고, 새로운 `custom_send_sms()` 같은 함수를 생성하여 기존 SMS 발송 트리거가 이 새로운 함수를 호출하도록 변경할 수도 있습니다. 이 방법은 기존 코드를 덜 건드려 안정성을 높일 수 있습니다.

*   **각 발송 트리거 파일:**
    *   회원가입 (`register_form_update.php`, `member_confirm.php` 등)
    *   주문 관련 (`shop/orderformupdate.php`, `shop/orderpartupdate.php` 등)
    *   관리자 페이지 (`adm/sms_form_update.php`, `adm/member_list_update.php` 등)
    *   위 파일들에서 `sms_lib.php`에 정의된 SMS 발송 함수를 호출하는 부분을 찾습니다. 해당 함수가 자체 API를 호출하도록 수정되었거나, 새로운 `custom_send_sms()` 함수를 호출하도록 변경합니다.

**2.3. 자체 API 연동 코드 작성 (예시)**

기존 SMS 라이브러리 파일(`lib/sms.lib.php` 등) 내에 다음과 같은 코드를 추가하거나 수정할 수 있습니다.

```php
<?php
// ... 기존 SMS 관련 코드 ...

// ----------------------------------------------------
// 자체 SMS API 연동을 위한 함수
// ----------------------------------------------------
function custom_send_sms($to_hp, $from_hp, $msg, $recv_dt = '') {
    global $config; // 전역 설정 변수 사용 (API 키 등을 여기서 가져올 수 있음)

    $api_url = $config['cf_custom_sms_api_url']; // 예: 설정에서 불러온 자체 API URL
    $api_key = $config['cf_custom_sms_api_key']; // 예: 자체 API 인증 키

    if (!$api_url || !$api_key) {
        // API 설정이 없으면 발송하지 않음 또는 에러 로그
        error_log('Custom SMS API URL or Key is not configured.');
        return false;
    }

    $data = array(
        'to' => $to_hp,
        'from' => $from_hp,
        'message' => $msg,
        'request_time' => date('Y-m-d H:i:s')
        // 필요한 경우 추가 파라미터 (예: 발송 요청 시간, 사용자 ID 등)
    );

    // JSON 형식으로 데이터 인코딩
    $json_data = json_encode($data);

    // cURL을 사용하여 자체 API 서비스로 요청 전송
    $ch = curl_init($api_url);
    curl_setopt($ch, CURLOPT_HTTPHEADER, array(
        'Content-Type: application/json',
        'X-API-KEY: ' . $api_key // 자체 API 인증 헤더 (설계에 따라 변경)
    ));
    curl_setopt($ch, CURLOPT_POST, 1);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $json_data);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_TIMEOUT, 5); // 타임아웃 5초

    $response = curl_exec($ch);
    $http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $curl_errno = curl_errno($ch);
    $curl_error = curl_error($ch);

    curl_close($ch);

    if ($curl_errno > 0) {
        error_log('Custom SMS API cURL Error (' . $curl_errno . '): ' . $curl_error);
        return false;
    }

    if ($http_code != 200) {
        error_log('Custom SMS API HTTP Error: ' . $http_code . ', Response: ' . $response);
        return false;
    }

    // 자체 API 서비스의 응답 형식에 따라 성공 여부 판단 로직 추가
    $result = json_decode($response, true);
    if (isset($result['status']) && $result['status'] === 'success') {
        return true; // 성공
    } else {
        error_log('Custom SMS API response indicates failure: ' . $response);
        return false; // 실패
    }
}
?>
```

**2.4. 기존 함수 호출 변경 (예시)**

기존에 `send_sms_icode($to_hp, $from_hp, $msg)` 와 같이 아이코드 함수를 호출하던 부분을 다음과 같이 변경합니다.

```php
// 기존 아이코드 호출 (주석 처리 또는 제거)
// include_once(G5_LIB_PATH.'/icode.sms.lib.php');
// $res = icode_send_sms($to_hp, $from_hp, $msg);

// 자체 API 호출
include_once(G5_LIB_PATH.'/sms.lib.php'); // 또는 커스텀 함수가 있는 파일
$res = custom_send_sms($to_hp, $from_hp, $msg);

if ($res) {
    // SMS 발송 성공
    // ...
} else {
    // SMS 발송 실패
    // ...
}
```

#### **3. 필요한 설정 변경 사항**

자체 API 연동을 위한 설정 정보(API URL, 인증 키 등)를 그누보드/영카트에 추가해야 합니다.

**3.1. `config.php` 또는 `_common.php`에 상수 정의**

간단하게는 `config.php` 파일에 직접 정의할 수 있습니다. (권장하지 않음, 업데이트 시 충돌 가능성)

```php
// config.php 또는 _common.php
define('G5_CUSTOM_SMS_API_URL', 'https://your.custom.sms.api/send');
define('G5_CUSTOM_SMS_API_KEY', 'YOUR_SECRET_API_KEY');
```
그리고 위에서 global $config 대신 G5_CUSTOM_SMS_API_URL, G5_CUSTOM_SMS_API_KEY 상수를 사용하도록 custom_send_sms 함수를 수정합니다.

**3.2. 데이터베이스 및 관리자 설정 페이지 추가 (권장)**

가장 유연하고 권장되는 방법은 데이터베이스에 새로운 설정 값을 추가하고, 관리자 페이지에서 이를 설정할 수 있도록 하는 것입니다.

*   **데이터베이스 `g5_config` 테이블 수정:**
    *   `cf_custom_sms_api_url`, `cf_custom_sms_api_key` 등의 컬럼을 추가하거나, `cf_sms_use` 와 같이 기존 SMS 설정 필드를 재활용하여 자체 API 사용 여부를 제어할 수 있습니다.
    *   SQL 쿼리 예시 (직접 DB에서 실행):
        ```sql
        ALTER TABLE g5_config ADD cf_custom_sms_api_url VARCHAR(255) NOT NULL DEFAULT '' AFTER cf_icode_server_port;
        ALTER TABLE g5_config ADD cf_custom_sms_api_key VARCHAR(255) NOT NULL DEFAULT '' AFTER cf_custom_sms_api_url;
        ALTER TABLE g5_config ADD cf_custom_sms_use TINYINT(4) NOT NULL DEFAULT '0' AFTER cf_custom_sms_api_key; -- 1: 자체 API 사용, 0: 기존 아이코드
        ```

*   **관리자 환경설정 페이지 수정:**
    *   `adm/config_form.php` 및 `adm/config_form_update.php` 파일을 수정하여 위에서 추가한 `cf_custom_sms_api_url`, `cf_custom_sms_api_key`, `cf_custom_sms_use` 값을 관리자 페이지에서 입력하고 저장할 수 있도록 폼 필드를 추가합니다.
    *   관리자 페이지에서 `cf_custom_sms_use` 값이 1일 때만 자체 API를 사용하고, 0일 때는 기존 아이코드 또는 다른 SMS 서비스를 사용하도록 조건부 로직을 구현할 수 있습니다.

#### **4. 고려사항 및 테스트**

*   **에러 처리 및 로깅:** 자체 API 호출 실패 시 어떻게 처리할 것인지 (예: 관리자에게 알림, 실패 로그 기록) 명확히 정의해야 합니다. `error_log()` 함수를 적극 활용하여 문제 발생 시 추적할 수 있도록 합니다.
*   **보안:** API 키와 같은 민감 정보는 코드에 직접 하드코딩하기보다 설정 파일이나 환경 변수, 데이터베이스를 통해 관리하는 것이 안전합니다. 자체 API 서비스 또한 적절한 인증 및 권한 부여 메커니즘을 갖추어야 합니다.
*   **비동기 처리:** 대량 문자 발송 시 웹 서버 부하를 줄이기 위해 SMS 발송을 비동기(Background Job)로 처리하는 것을 고려할 수 있습니다. (예: 큐 시스템 연동)
*   **테스트:** 코드 수정 후에는 회원가입, 주문 등 SMS 발송이 이루어지는 모든 시나리오에서 문자가 정상적으로 발송되는지 철저히 테스트해야 합니다.

---