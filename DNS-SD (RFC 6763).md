---
share: true  
---


# 3. Design Goals

1. 특정 logical domain에서 특정 service type에 대한 질의 기능 및 이에 대한 응답으로 named instance를 주어야 함 (network browsing | **`Service Instance Enumeration`**)))
2. 특정 named instance가 주어졌을 때, 해당 이름으로 효율적인 질의를 가능하게 해야 함 (IP 주소, 포트 번호 등)
3. Instance의 이름은 상대적으로 안정적으로 유지되어야 함
	- 프린터를 한 번 찾았다면 설사 프린터의 물리적 주소 (e.g., IP address)가 변경되더라도 유저가 1번부터의 과정을 반복할 필요가 없어야 함

- Service discovery에 대한 더 자세한 requirement는 [RFC 6760](https://datatracker.ietf.org/doc/html/rfc6760) 참고


# 4. Service Instance Enumeration (Browsing)

서비스 instance를 나열하는 (찾는) 방법에 관한 기술임.

- 기본적으로 DNS SRV 레코드[(RFC 2782)](https://datatracker.ietf.org/doc/html/rfc2782)에 기반함 
	- 단, 기존의 DNS SRV 레코드는 해당 레코드에서 제공하는 server들이 모두 같은 서비스를 제공한다고 간주하고 DNS SRV specification에 따라 (e.g., weight, priority 규칙) 그 중 하나에 접근하면 된다고 봄.
	- 그러나, 이 문서에서 다루는 service dicovery의 관점에서는 모든 서버가 동일하지 않음. 예를 들어, 워드 프로세서 프로그램에서 인쇄를 위해 레코드를 찾을 경우, 회사 내의 아무 프린터에나 접근해 인쇄한다면 이는 사용자의 필요를 만족시킬 수 없음.
	- 따라서, DNS-SD에서는 PTR 레코드라는 하나의 indirection을 두는 방식을 취함.
		- 

## Structured Service Instance Names

위에서 언급한 바와 같이 DNS-SD에서는 PTR 레코드를 검색함. 
- PTR 레코드의 검색은 \<Service\>.\<Domain\>에 대해서 이뤄지며 그 결과는 **Service Instance Name**을 **0개 이상** 반환함 
	- Service Instance Name  = \<Instance\> . \<Service\> . \<Domain\>
	- Instance Name은 user-friendly하게 지어져야 하며 default 값의 경우도 

1. **Instance Name**: service를 제공하는 instance를 파악하기 위한 이름(?)
	- User-friendly하게 지어져야 함. (구체적인 제한 조건등은 필요하다면 문서 참고)
2. **Service Name**: pair of DNS labels 
	- 첫째 라벨: (underscore + service name)
		- service가 어떤 일을 하는 지, 어떤 application protocol이 사용되는 지를 명시 (구체적인 내용은 Section 7 참고)
	- 둘째 라벨: \_tcp | \_udp 
3. **Domain Name**: DNS subdomain을 명시
	- local. : link-local Multicast DNS (RFC 6762)를 쓰거나, 일반적인 Unicast DNS domain name 사용

## 기타 (4.2 ~ 4.3)
- User Interface Presentation: 
	- 검색하면 결과 여러 개 나오니까 알아서 잘 보여줄 것 
	- 일반적으로 service와 domain은 알고 있다고 가정하여 instance name만 보여주는 것도 가능하나 반드시 그런 것은 아니므로 주의 필요 (e.g., 다른 domain 내의 서비스 검색, subtype (?) 등)
- Name handling: 3개 종류의 이름을 붙여서 쓸 경우, instance name은 아무 글자나 포함 가능하므로 dot이나 backslash 문자를 escape 해줄 필요가 있음


# 5. Service Instance Resoluation

- 원하는 서비스의 instance를 찾았을 경우, 해당 instance의 SRV 레코드와 TXT 레코드를 다시 질의함
	- SRV 레코드: 포트 및 host name
	- TXT 레코드: 추가적인 정보 제공 (section 6 참고)
- 대부분의 경우, 한 서비스 instance에 대해서 하나의 레코드가 돌아올 것이며 이 경우 weight 및 priority 값은 0으로 설정
	- 단, 여러 개가 돌아오는 경우 적절하게 값에 따라 행동해야 함 (lower priority가 우선권, 같은 priority끼리 weight에 따라 random)


# 6. Data Syntax for DNS-SD TXT Records

* DNS-SD에서는 서비스에 대한 별도의 정보가 필요한 경우 이를 TXT에 레코드에 저장 및 배포하도록 되어 있음
	* e.g., 파일 서비스의 볼륨 등
	* TXT 레코드는 설사 별도의 정보가 없더라도 (zero-byte여도) 반드시 저장되어야만 함.
	* Note: 이 제약은 DNS-SD의 경우에만 적용되는 것으로 일반적인 SRV 레코드에 해당하는 것은 아님.
* 레코드 길이
	* 65535 bytes까지의 길이를 가질 수 있으며 (note: mDNS와 같이 사용할 경우 mDNS의 9000byte 제약이 더 먼저임.) 
	* 1개 이상의 string으로 구성 
		* single length byte + 0~255bytes text
		* 정보가 없을 경우 원칙적으로는 1개의 zero byte를 가지는 string을 보내야 함
		* string에 관한 구체적인 format은 (RFC 1035 참고)
	* 일반적으로는 약 200byte 안의 길이를 가지는 것을 의도함


+ ### 6.3 TXT 레코드는 key-value store
	+ 각 string은 key=value store의 형태. 첫 `=`를 만날 때까지는 key, 그 이후는 value (다른 `=` 글자가 있는 경우 해당 글자를 포함해서라도)
	+ port number나 host name 다시 넣지 말 것


- ### 6.4 key에 대한 규정
	- at least 1 character, no more than 9 characters
	- 서비스의 context 안에서 unique, unambiguous해야 함 (만약 여러 개의 같은 key가 있다면 첫째 빼고 뒤는 무시)
	- 기본적으로 machine-readable한 name을 의도
	- 대소문자는 구분 없음


+ ### 6.5 value에 대한 규정
	+ key 다음의 `=` 뒤에 오는 모든 내용은 value
	+ 텍스트 데이터 (utf-8, ascii 등에 기반한)일 수 있으나 기본적으로 임의의 binary data를 가정

- ### 6.6 TXT 레코드 예시
![[DNS-TXT example.png|640]]

- ### 6.7 version tag
	- 첫 key/value로 txtvers=# (#은 decimal version number)를 포함할 것을 권장
	- key/value들의 version에 대한 정보이지 application protocol의 version number는 아니라고 간주 (해당 경우는 protovers라는 key를 권장)


# 7. Service Names

앞서 언급한 바와 같이 2개의 label로 이루어짐

1. underscore + service protocol
	- service protocol name에 대한 구체적인 rule은 RFC 6335 참고
2. \_tcp or \_udp
	- \_tcp 기반 서비스에는 모두 tcp를 그렇지 않다면 udp를 사용 (이는 DNS-SD에만 적용되는 semantic임)

## 7.1 Selective Instance Enumeration (Subtypes)

검색 범위를 줄이고 싶은 경우가 있을 수 있음

# 8. Flagship Naming

동일한 logical 기능을 제공하는 여러 service가 있을 경우에 대해 그 중 하나의 protocol을 flagship으로 지정해야 함
- 크게 중요한 내용은 아닌 듯함
# 9. Service Type Enumeration

일반적인 사용 예시에서는 client가 network내의 모든 service를 찾을 필요는 없을 것임.
- 만약 그런 경우와 관련해 정보가 필요할 경우 원본 문서 참고
# 10. Populating the DNS with Information

DNS에 레코드를 어떻게 채울 지는 이 문서의 scope가 아니며 여기서는 간단한 예시만을 제공
- Administarator가 수동으로 등록
- Network monitoring tool이 scan해서 걸리는 장비를 등록
- Service Manager (e.g., printer manger device)가 정보를 등록
	- 혹은 subdomain의 일부를 delegate 받아서 직접 처리
- **zeroconf** (이게 구현체에서 많이 등장하던데 정확히 어떤 개념인가?)


# 11. Discovery of Browsing and Registration Domains (Domain Enumeration)

새로운 user client가 service를 network 내에서 가능한 service를 **자동으로** 어떻게 찾을 것인가?
- 아래와 같은 미리 할당된 RR name을 사용
	- b.\_dns-sd.\_udp.\<domain\>. : list of recommended domains for browsing
	- db.\_dns-sd.\_udp.\<domain>. : single recommended default domain for browsing
	- r.\_dns-sd.\_udp.\<domain>. : list of domains recommended for registering services using
      Dynamic Update.
	- dr.\_dns-sd.\_udp.\<domain>. : single recommended default domain for registering services.
	- lb.\_dns-sd.\_udp.\<domain>.