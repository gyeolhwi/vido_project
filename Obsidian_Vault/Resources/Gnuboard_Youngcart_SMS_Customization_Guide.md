# 그누보드 및 영카트 자체 SMS API 연동 가이드 (개념적 접근)

이 가이드는 그누보드 및 영카트의 일반적인 구조와 외부 API 연동 패턴을 기반으로 작성되었습니다. 웹 검색 제한으로 인해 실제 시스템과 다를 수 있으니, 적용 전 반드시 백업하고 신중하게 작업하십시오.

---

## 1. 현재 아이코드 연동 방식 분석 (추정)

그누보드 및 영카트는 외부 문자 발송 서비스 연동 시 다음 패턴을 따릅니다:

*   **SMS 라이브러리 파일:** `lib/sms.lib.php` 또는 `extend/sms_extend.php` 같은 핵심 파일에 아이코드 API 호출 로직이 포함될 가능성이 높습니다.
*   **설정 파일:** `config.php`, `_common.php` 또는 `data/sms_conf.php` 등에서 아이코드 연동에 필요한 API 키, ID, 비밀번호 정보가 정의되거나 데이터베이스에서 불러와집니다.
*   **문자 발송 트리거:** 회원가입, 주문 완료, 관리자 발송 등 특정 이벤트 발생 시 SMS 라이브러리/클래스의 함수를 호출하여 문자를 보냅니다.
    *   예시: `register_form_update.php`, `shop/orderformupdate.php`, `adm/sms_form_update.php`

현재 아이코드 연동은 주로 핵심 라이브러리 내에서 아이코드 PHP SDK를 사용하거나, HTTP(POST/GET) 요청으로 직접 아이코드 API 엔드포인트에 데이터를 전송하는 방식으로 구현될 것입니다.

## 2. 자체 API 연동을 위한 코드 수정

기존 아이코드 API 호출 로직을 찾아 자체 API 호출 로직으로 변경해야 합니다.

### 2.1. 자체 API 서비스 인터페이스 설계

자체 API 서비스가 어떤 엔드포인트로 어떤 형식의 데이터를 받아 문자를 발송할지 먼저 설계합니다.
*   **예시:** `POST /api/sms/send`, 본문: `{"to": "010-1234-5678", "message": "안녕하세요.", "from": "02-123-4567"}`

### 2.2. 핵심 수정 지점

SMS 발송이 이루어지는 **핵심 함수 또는 클래스**를 수정합니다.

*   **`lib/sms.lib.php` (또는 유사 파일):**
    *   아이코드 관련 함수(예: `icode_send_sms()`, `send_sms_icode()`)를 찾아 내부의 아이코드 API 호출 부분을 자체 API 호출 로직으로 변경합니다.
    *   **대안:** 기존 아이코드 함수는 유지하고, `custom_send_sms()` 같은 새 함수를 만들어 기존 SMS 발송 트리거가 이 함수를 호출하도록 변경하면 기존 코드 변경을 최소화하여 안정성을 높일 수 있습니다.

*   **각 발송 트리거 파일:**
    *   회원가입, 주문, 관리자 페이지 등에서 SMS 발송 함수를 호출하는 부분을 찾습니다. (예: `register_form_update.php`, `shop/orderformupdate.php`, `adm/sms_form_update.php`)
    *   해당 함수가 자체 API를 호출하도록 수정되었거나, `custom_send_sms()` 함수를 호출하도록 변경합니다.

### 2.3. 자체 API 연동 코드 작성 (예시)

기존 SMS 라이브러리 파일(`lib/sms.lib.php` 등)에 다음 PHP 코드를 추가하거나 수정합니다.

```php
<?php
// ... 기존 SMS 관련 코드 ...

// ----------------------------------------------------
// 자체 SMS API 연동 함수
// ----------------------------------------------------
function custom_send_sms($to_hp, $from_hp, $msg, $recv_dt = '') {
    global $config; // 전역 설정 변수 사용

    $api_url = $config['cf_custom_sms_api_url']; // 설정에서 불러온 자체 API URL
    $api_key = $config['cf_custom_sms_api_key']; // 자체 API 인증 키

    if (!$api_url || !$api_key) {
        error_log('Custom SMS API URL 또는 Key가 설정되지 않았습니다.');
        return false;
    }

    $data = array(
        'to' => $to_hp,
        'from' => $from_hp,
        'message' => $msg,
        'request_time' => date('Y-m-d H:i:s')
        // 필요한 경우 추가 파라미터 포함
    );

    $json_data = json_encode($data); // JSON 형식으로 데이터 인코딩

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
        error_log('Custom SMS API cURL 오류 (' . $curl_errno . '): ' . $curl_error);
        return false;
    }
    if ($http_code != 200) {
        error_log('Custom SMS API HTTP 오류: ' . $http_code . ', 응답: ' . $response);
        return false;
    }

    $result = json_decode($response, true);
    if (isset($result['status']) && $result['status'] === 'success') {
        return true; // 성공
    } else {
        error_log('Custom SMS API 응답 실패: ' . $response);
        return false; // 실패
    }
}
?>
```

### 2.4. 기존 함수 호출 변경 (예시)

기존 `send_sms_icode($to_hp, $from_hp, $msg)` 호출 부분을 다음과 같이 변경합니다.

```php
// 기존 아이코드 호출 (주석 처리 또는 제거)
// include_once(G5_LIB_PATH.'/icode.sms.lib.php');
// $res = icode_send_sms($to_hp, $from_hp, $msg);

// 자체 API 호출
include_once(G5_LIB_PATH.'/sms.lib.php'); // 또는 커스텀 함수가 있는 파일
$res = custom_send_sms($to_hp, $from_hp, $msg);

if ($res) {
    // SMS 발송 성공 처리
} else {
    // SMS 발송 실패 처리
}
```

## 3. 필요한 설정 변경 사항

자체 API 연동을 위한 설정 정보(API URL, 인증 키 등)를 그누보드/영카트에 추가해야 합니다.

### 3.1. `config.php` 또는 `_common.php`에 상수 정의 (간단한 방법)

간단하게 `config.php` 파일에 직접 정의할 수 있습니다. (업데이트 시 충돌 가능성으로 권장하지 않음)

```php
// config.php 또는 _common.php
define('G5_CUSTOM_SMS_API_URL', 'https://your.custom.sms.api/send');
define('G5_CUSTOM_SMS_API_KEY', 'YOUR_SECRET_API_KEY');
```
`custom_send_sms` 함수에서는 `global $config;` 대신 이 상수들을 직접 사용하도록 수정합니다.

### 3.2. 데이터베이스 및 관리자 설정 페이지 추가 (권장)

가장 유연하고 권장되는 방법은 데이터베이스에 새로운 설정 값을 추가하고, 관리자 페이지에서 이를 설정할 수 있도록 하는 것입니다.

*   **데이터베이스 `g5_config` 테이블 수정:**
    *   `cf_custom_sms_api_url`, `cf_custom_sms_api_key` 등의 컬럼을 추가합니다.
    *   `cf_custom_sms_use` 컬럼을 추가하여 자체 API 사용 여부를 제어할 수 있습니다. (1: 자체 API 사용, 0: 기존 아이코드)
    *   **SQL 쿼리 예시 (직접 DB에서 실행):**
        ```sql
        ALTER TABLE g5_config ADD cf_custom_sms_api_url VARCHAR(255) NOT NULL DEFAULT '' AFTER cf_icode_server_port;
        ALTER TABLE g5_config ADD cf_custom_sms_api_key VARCHAR(255) NOT NULL DEFAULT '' AFTER cf_custom_sms_api_url;
        ALTER TABLE g5_config ADD cf_custom_sms_use TINYINT(4) NOT NULL DEFAULT '0' AFTER cf_custom_sms_api_key;
        ```

*   **관리자 환경설정 페이지 수정:**
    *   `adm/config_form.php` 및 `adm/config_form_update.php` 파일을 수정하여 `cf_custom_sms_api_url`, `cf_custom_sms_api_key`, `cf_custom_sms_use` 값을 관리자 페이지에서 입력하고 저장할 수 있도록 폼 필드를 추가합니다.
    *   `cf_custom_sms_use` 값이 1일 때만 자체 API를 사용하고, 0일 때는 기존 아이코드 또는 다른 SMS 서비스를 사용하도록 조건부 로직을 구현합니다.

## 4. 고려사항 및 테스트

*   **에러 처리 및 로깅:** 자체 API 호출 실패 시 관리자 알림, 실패 로그 기록 등 명확한 에러 처리 정의가 필요합니다. `error_log()` 함수를 활용하여 문제 발생 시 추적합니다.
*   **보안:** API 키 등 민감 정보는 코드 하드코딩보다 설정 파일, 환경 변수, 데이터베이스를 통해 안전하게 관리해야 합니다. 자체 API 서비스도 적절한 인증 및 권한 부여가 필수입니다.
*   **비동기 처리:** 대량 문자 발송 시 웹 서버 부하를 줄이기 위해 SMS 발송을 비동기(Background Job)로 처리하는 방안을 고려합니다. (예: 큐 시스템 연동)
*   **테스트:** 코드 수정 후에는 회원가입, 주문 등 SMS 발송이 이루어지는 모든 시나리오에서 문자가 정상적으로 발송되는지 철저히 테스트해야 합니다.
