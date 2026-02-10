# 그누보드 & 영카트: 자체 SMS API 연동 가이드 (개념적 접근)

이 가이드는 그누보드 및 영카트의 일반적인 구조와 외부 SMS API 연동 패턴을 바탕으로 합니다.
웹 검색 제한으로 인해 실제 환경과 다를 수 있으므로, **적용 전 반드시 시스템을 백업하고 신중하게 작업하십시오.**

---

## 1. 현재 아이코드 SMS 연동 방식 파악 (추정)

그누보드와 영카트에서 아이코드와 같은 외부 SMS 서비스는 다음 방식으로 연동됩니다:

*   **핵심 라이브러리 파일:**
    *   `lib/sms.lib.php` 또는 `extend/sms_extend.php` 등 SMS 관련 핵심 로직을 처리하는 파일에 아이코드 API 호출 코드가 포함되어 있을 것입니다.
*   **설정 파일:**
    *   `config.php`, `_common.php`, 또는 `data/sms_conf.php` 등에서 아이코드 연동에 필요한 **API 키, ID, 비밀번호**와 같은 정보가 정의되거나 데이터베이스에서 불러옵니다.
*   **문자 발송 트리거:**
    *   회원가입, 주문 완료, 관리자 문자 발송 등 특정 이벤트 발생 시, 위의 SMS 라이브러리 파일 내 함수를 호출하여 문자를 발송합니다.
    *   **주요 트리거 파일 예시:**
        *   `register_form_update.php` (회원가입)
        *   `shop/orderformupdate.php` (주문 관련)
        *   `adm/sms_form_update.php` (관리자 발송)

아이코드 연동은 대개 이 핵심 라이브러리 파일 내에서 아이코드 PHP SDK를 사용하거나, **HTTP(POST/GET) 요청**을 통해 아이코드 API 엔드포인트에 직접 데이터를 전송하는 방식으로 구현되어 있을 것으로 예상됩니다.

## 2. 자체 SMS API 연동을 위한 코드 수정 방안

기존 아이코드 API 호출 로직을 찾아 자체 API 호출 로직으로 교체해야 합니다.

### 2.1. 자체 API 서비스 인터페이스 설계

자체 API 서비스가 어떤 **엔드포인트(URL)**를 통해 어떤 **데이터 형식(JSON 등)**을 받아 문자를 발송할지 먼저 정의해야 합니다.

*   **API 호출 예시:**
    *   **메서드:** `POST`
    *   **URL:** `/api/sms/send`
    *   **요청 본문 (JSON):**
        ```json
        {
          "to": "010-1234-5678",
          "message": "안녕하세요.",
          "from": "02-123-4567",
          "request_time": "2026-02-10 23:46:00"
        }
        ```

### 2.2. 핵심 코드 수정 지점

SMS 발송이 실제로 이루어지는 **핵심 함수 또는 클래스**를 수정합니다.

*   **`lib/sms.lib.php` (또는 유사 SMS 라이브러리 파일):**
    *   **수정 목표:** 아이코드 관련 함수(예: `icode_send_sms()`, `send_sms_icode()`) 내부의 **아이코드 API 호출 부분을 자체 API 호출 로직으로 변경**합니다.
    *   **권장 대안:** 기존 아이코드 함수는 그대로 두고, `custom_send_sms()`와 같은 **새로운 함수를 생성**합니다. 이후 기존 SMS 발송 트리거들이 이 `custom_send_sms()`를 호출하도록 변경하면, 기존 코드를 덜 건드려 안정성을 높일 수 있습니다.

*   **각 문자 발송 트리거 파일:**
    *   회원가입, 주문, 관리자 페이지 등에서 SMS 발송 함수를 호출하는 부분을 찾습니다.
    *   해당 함수가 **자체 API를 호출하도록 수정되었거나, 새로 만든 `custom_send_sms()` 함수를 호출**하도록 변경합니다.

### 2.3. 자체 API 연동 PHP 코드 예시

기존 SMS 라이브러리 파일(`lib/sms.lib.php` 등)에 다음 PHP 코드를 추가하거나 수정할 수 있습니다.

```php
<?php
// ... 기존 SMS 관련 코드 ...

/**
 * 자체 SMS API 연동을 위한 문자 발송 함수
 *
 * @param string $to_hp    수신자 휴대폰 번호
 * @param string $from_hp  발신자 휴대폰 번호
 * @param string $msg      발송 메시지 내용
 * @param string $recv_dt  (선택 사항) 예약 발송 시간
 * @return bool            발송 성공 시 true, 실패 시 false
 */
function custom_send_sms($to_hp, $from_hp, $msg, $recv_dt = '') {
    global $config; // 그누보드/영카트 전역 설정 변수 사용

    // 자체 API URL 및 인증 키를 설정에서 불러옵니다.
    $api_url = $config['cf_custom_sms_api_url'];
    $api_key = $config['cf_custom_sms_api_key'];

    // API 설정이 없으면 발송하지 않고 오류를 기록합니다.
    if (!$api_url || !$api_key) {
        error_log('오류: 자체 SMS API URL 또는 Key가 설정되지 않았습니다.');
        return false;
    }

    // 자체 API로 전송할 데이터 구조
    $data = array(
        'to' => $to_hp,
        'from' => $from_hp,
        'message' => $msg,
        'request_time' => date('Y-m-d H:i:s')
        // 필요에 따라 추가 파라미터(예: 사용자 ID, 발송 유형 등)를 포함할 수 있습니다.
    );

    $json_data = json_encode($data); // 데이터를 JSON 형식으로 인코딩

    // cURL을 사용하여 자체 API 서비스로 HTTP POST 요청 전송
    $ch = curl_init($api_url);
    curl_setopt($ch, CURLOPT_HTTPHEADER, array(
        'Content-Type: application/json',
        'X-API-KEY: ' . $api_key // 자체 API 인증을 위한 커스텀 헤더 (API 설계에 따라 변경)
    ));
    curl_setopt($ch, CURLOPT_POST, 1); // POST 요청으로 설정
    curl_setopt($ch, CURLOPT_POSTFIELDS, $json_data); // 요청 본문에 JSON 데이터 포함
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true); // 응답을 문자열로 반환
    curl_setopt($ch, CURLOPT_TIMEOUT, 5); // 5초 후 타임아웃

    $response = curl_exec($ch); // API 요청 실행
    $http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE); // HTTP 응답 코드
    $curl_errno = curl_errno($ch); // cURL 오류 번호
    $curl_error = curl_error($ch); // cURL 오류 메시지

    curl_close($ch); // cURL 세션 종료

    // cURL 통신 오류 처리
    if ($curl_errno > 0) {
        error_log('자체 SMS API cURL 통신 오류 (' . $curl_errno . '): ' . $curl_error);
        return false;
    }
    // HTTP 응답 코드 오류 처리 (예: 200 OK가 아닌 경우)
    if ($http_code != 200) {
        error_log('자체 SMS API HTTP 오류: ' . $http_code . ', 응답: ' . $response);
        return false;
    }

    // 자체 API 서비스의 응답 형식에 따라 성공 여부를 판단합니다.
    $result = json_decode($response, true);
    if (isset($result['status']) && $result['status'] === 'success') {
        return true; // SMS 발송 성공
    } else {
        error_log('자체 SMS API 응답에서 실패 확인: ' . $response);
        return false; // SMS 발송 실패
    }
}
?>
```

### 2.4. 기존 SMS 발송 함수 호출 변경 (예시)

기존에 아이코드 함수(예: `send_sms_icode($to_hp, $from_hp, $msg)`)를 호출하던 부분을 다음과 같이 변경합니다.

```php
// 기존 아이코드 호출 (주석 처리 또는 제거)
// include_once(G5_LIB_PATH.'/icode.sms.lib.php');
// $res = icode_send_sms($to_hp, $from_hp, $msg);

// 자체 SMS API 호출 (새롭게 정의한 함수 사용)
include_once(G5_LIB_PATH.'/sms.lib.php'); // 'custom_send_sms' 함수가 정의된 파일
$res = custom_send_sms($to_hp, $from_hp, $msg);

if ($res) {
    // SMS 발송 성공 시 추가 로직
} else {
    // SMS 발송 실패 시 추가 로직
}
```

## 3. 필요한 설정 변경 사항

자체 API 연동에 필요한 정보(API URL, 인증 키 등)를 그누보드/영카트 시스템에 추가해야 합니다.

### 3.1. `config.php` 또는 `_common.php`에 상수 정의 (간단하지만 권장하지 않음)

가장 간단한 방법은 `config.php` 파일에 직접 API 정보를 상수로 정의하는 것입니다.
**단점:** 시스템 업데이트 시 충돌 가능성이 있어 **권장하지 않습니다.**

```php
// config.php 또는 _common.php 파일 내
define('G5_CUSTOM_SMS_API_URL', 'https://your.custom.sms.api/send');
define('G5_CUSTOM_SMS_API_KEY', 'YOUR_SECRET_API_KEY_HERE'); // 실제 키로 변경
```
*주의:* `custom_send_sms` 함수 내에서 `global $config;` 대신 이 상수들을 직접 사용하도록 코드를 수정해야 합니다.

### 3.2. 데이터베이스 및 관리자 설정 페이지 추가 (권장)

가장 유연하고 관리하기 좋은 방법은 데이터베이스에 새로운 설정 값을 추가하고, 관리자 페이지에서 이를 쉽게 설정할 수 있도록 하는 것입니다.

*   **데이터베이스 `g5_config` 테이블 수정:**
    *   다음 SQL 쿼리를 데이터베이스에서 직접 실행하여 새로운 컬럼들을 추가합니다.
    *   `cf_custom_sms_api_url`: 자체 API 서비스의 URL
    *   `cf_custom_sms_api_key`: 자체 API 서비스의 인증 키
    *   `cf_custom_sms_use`: 자체 API 사용 여부 (1: 사용, 0: 사용 안 함)
    *   **SQL 쿼리 예시:**
        ```sql
        ALTER TABLE g5_config ADD cf_custom_sms_api_url VARCHAR(255) NOT NULL DEFAULT '' AFTER cf_icode_server_port;
        ALTER TABLE g5_config ADD cf_config ADD cf_custom_sms_api_key VARCHAR(255) NOT NULL DEFAULT '' AFTER cf_custom_sms_api_url;
        ALTER TABLE g5_config ADD cf_custom_sms_use TINYINT(4) NOT NULL DEFAULT '0' AFTER cf_custom_sms_api_key;
        ```

*   **관리자 환경설정 페이지 수정:**
    *   `adm/config_form.php` (설정 폼) 및 `adm/config_form_update.php` (설정 저장 처리) 파일을 수정합니다.
    *   위에서 추가한 `cf_custom_sms_api_url`, `cf_custom_sms_api_key`, `cf_custom_sms_use` 값을 관리자 페이지에서 입력하고 저장할 수 있도록 **폼 필드를 추가**합니다.
    *   관리자 페이지에서 `cf_custom_sms_use` 값이 `1`로 설정되었을 때만 자체 API를 사용하고, `0`일 때는 기존 아이코드 또는 다른 SMS 서비스를 사용하도록 **조건부 로직을 구현**합니다.

## 4. 중요 고려사항 및 철저한 테스트

*   **에러 처리 및 로깅:**
    *   자체 API 호출 실패 시 어떻게 처리할지 (예: 관리자에게 알림, 실패 로그 기록) 명확히 정의해야 합니다.
    *   `error_log()` 함수를 적극 활용하여 문제 발생 시 신속하게 추적할 수 있도록 합니다.
*   **보안:**
    *   API 키와 같은 민감 정보는 코드에 직접 하드코딩하기보다 **설정 파일, 환경 변수 또는 데이터베이스를 통해 관리**하는 것이 안전합니다.
    *   자체 API 서비스 또한 적절한 **인증 및 권한 부여 메커니즘**을 반드시 갖추어야 합니다.
*   **비동기 처리:**
    *   대량 문자 발송 시 웹 서버의 부하를 줄이기 위해 SMS 발송을 **비동기(Background Job)**로 처리하는 방안을 고려할 수 있습니다. (예: 큐 시스템 연동)
*   **철저한 테스트:**
    *   코드 수정 후에는 **회원가입, 주문, 관리자 문자 발송** 등 SMS 발송이 이루어지는 **모든 시나리오**에서 문자가 정상적으로 발송되는지 **꼼꼼하게 테스트**해야 합니다.
