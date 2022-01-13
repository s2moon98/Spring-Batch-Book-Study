# ItemReader
* 청크 기반으로 동작하는 각 스텝은
    * ItemReader, ItemProcessor, ItemWriter로 구성
* 스프링 배치는 거의 모든 유형의 입력 데이터를 처리할 수 있는 표준 방법을 제공
    * 추가로 커스텀 리 개발 기능도 제공

### ItemReader 인터페이스
* org.springframework.batch.item.ItemReader<T> 인터페이스
    * 스텝에 입력을 제공할 때 사용하는 단일 메서드 read를 제공
    ```JAVA
    public interface ItemReader {
        T read() throws Exception, UnexpectedinputException, ParseException,
                        NonTransientResourceException;
    }
    ```
    * ItemReader 인터페이스는 전략 인터페이스 - 처리할 입력 유형에 맞는 구현체를 제공
        * 플랫 파일, 여러 데이터베이스, JMS 리소스, 기타 입력 소스 등

### 파일 입력
* 스프링 배치가 제공하는 여러 선언적인 리더를 알아봅니다.

#### 플랫 파일
* 한개 또는 그 이상의 레코드가 포함된 특정 파일로 파일 내에 데이터의 포맷이나 의미를 정의하는 메타데이터가 없는 파일
* 스프링 배치에서 파일을 읽을 때 사용하는 컴포넌트를 먼저 알아봅니다.
    * FlatfileItemReader의 컴포넌트 - P252 그림 7-1
        * 다음 두 개의 메인 컴포넌트로 구성 된다.
            * 읽어들일 대상 파일을 나타내는 스프링의 Resource
            * org.springframework.batch.item.file.Linemapper 인터페이스 구현체
        * 자주 사용하는 애트리뷰트
            * P252 vy 7-1 FlatFileItemReader 구성 옵션
        * 파일을 읽을때는 파일에서 레코드 한 개에 해당하는 문자열이 LineMapper 구현체에 전달된다.
        * 가장 많이 사용되는 구현체는 DefaultLineMapper
            * 파일에서 읽은 원시 String을 다음 두 단계를 거쳐 사용할 도메인 객체로 변환
                * LineTokenizer
                    * 해당 줄을 파싱해 FiledSet으로 만든다.
                * FieldSetMapper
                    * FieldSet을 도메인 객체로 매핑
* 고정 너비 파일
    * 레거시 메인프레임 시스템에서는 고정 너비 파일을 사용하는 것이 일반적
    
#### XML
