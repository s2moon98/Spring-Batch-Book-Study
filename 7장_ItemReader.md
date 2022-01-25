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
## 여러줄에 걸친 레코드
스프링 배치가 공식적으로 제공하는 예제에서는 레코드가 끝나는 부분을 파악하기 위해 footer를 제공하지만 없는 레코드가 훨씬 많다. 이를 해결하기 위해서 customerItemReader 빈을 커스텀한다.

> 1. read 메서드가 Customer 객체를 이미 읽어들였는지 확인
>
>2-1. 읽어들이지 않았다면 FlatFileItemReader에서 읽기 시도
>
>3-1. 읽었다면 Customer 객체 내부의 Transaction List를 초기화
>3-2. 다음 읽은 레코드가 Transaction이면 Customer 객체의 Transaction list에 추가

```java
public class CustomerFileReader implements ItemStreamReader<Customer> {

	private Object curItem = null;

	private ItemStreamReader<Object> delegate;

	public CustomerFileReader(ItemStreamReader<Object> delegate) {
		this.delegate = delegate;
	}

	public Customer read() throws Exception {
		if(curItem == null) {
			curItem = delegate.read();
		}

		Customer item = (Customer) curItem;
		curItem = null;

		// 제어 중지 로직
		if(item != null) {
			item.setTransactions(new ArrayList<>());

			while(peek() instanceof Transaction) {
				item.getTransactions().add((Transaction) curItem);
				curItem = null;
			}
		}

		return item;
	}

	// 레코드를 미리 읽어놓는다.
	private Object peek() throws Exception {
		if (curItem == null) {
        		// 레코드가 처리 됐으면 다음 레코드 읽기
            		curItem = delegate.read();
		}
        
        	// 캐시 변수에 저장
		return curItem;
	}

	public void close() throws ItemStreamException {
		delegate.close();
	}

	public void open(ExecutionContext arg0) throws ItemStreamException {
		delegate.open(arg0);
	}

	public void update(ExecutionContext arg0) throws ItemStreamException {
		delegate.update(arg0);
	}
}
```
ItemReader 인터페이스가 아닌 IteamStreamReader 인터페이스를 구현했다. 
-> ItemReader를 사용하면 ExecutionContext도 관리해야 하기 때문에 이에 대한 구현도 필요하기 때문이다.

이제 customItemReader를 사용하려면 delegate 객체의 의존성이 필요하다.

```java
	@Bean
	public CustomerFileReader customerFileReader() {
		return new CustomerFileReader(customerItemReader(null));
	}
```

customItemReader를 감싼 customFileReader 빈을 선언한다. 해당 빈을 step의 reader로 지정하고 step을 실행하면 결과가 잘 출력된 것을 볼 수 있다.

## 여러개의 소스

`MultiResourceItemReader` 이용.
다른 `ItemReader`를 래핑하는데 이 때 읽어야 할 파일명의 패턴을 `MultiResourceItemReader`의 의존성으로 정의한다.

```java
	@Bean
	@StepScope
	public MultiResourceItemReader multiCustomerReader(@Value("#{jobParameters['customerFile']}")Resource[] inputFiles) {	
		return new MultiResourceItemReaderBuilder<>()
				.name("multiCustomerReader")		// reader 이름
				.resources(inputFiles)		// resource 객체 배열
				.delegate(customerFileReader())		// 실제 작업할 위임 컴포넌트
				.build();
	}

        @Bean
	public CustomerFileReader customerFileReader() {
		return new CustomerFileReader(customerItemReader());
	}
   
	@Bean
	@StepScope
	public FlatFileItemReader customerItemReader() {
        	// resource를 구성하는 코드 제거
        	// resource 객체 배열 이용하기 때문에 
        	return new FlatFileItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.lineMapper(lineTokenizer())
				.build();
	}
    
    
```

```java
public class CustomerFileReader 
	// 모든 ItemReader를 지원하기 위해 상속
	implements ResourceAwareItemReaderItemStream<Customer> {

.
.
.
	// resource 주입하기 위함 
	@Override
	public void setResource(Resource resource) {
		this.delegate.setResource(resource);
	}
}

```
ItemReader에 복수개의 resource를 주입하기 위해서 2가지를 변경한다.

1. `FlatFileItemReader`에서 resource 구성하는 코드 제거

2. `customerFileReader`도 `ResourceAwareItemReaderItemStream` 인터페이스를 구현하도록 수정
해당 인터페이스의 목적은 모든 ItemReader를 지원하는 것이다.
해당 인터페이스의 setResource 메서드를 구현해서 resource를 주입할 수 있도록 한다.

resource를 주입할 수 있게 되면 ItemReader가 스스로 파일 관리는 하는 대신에 필요한 각 파일을 스프링 배치가 생성해 ItemReader에게 주입해줄 수 있다.

하지만 위와 같은 예제는 재시작같은 상황에서 추가적인 안전장치를 제공하지 않는다. 예를 들면 1, 2, 3 파일 실행 중에 2 파일에서 오류가 나서 재시작하는 사이에 4 파일이 추가될 경우 1, 2, 3, 4 파일을 실행한다. 이를 방지하기 위해서 새로 생성한 파일을 새로운 디렉터리에 넣는다.

## JSON
JsonObjectReader는 Jackson, Gson을 이용하는 두가지 인터페이스 구현체를 제공한다. Jackson을 이용하는 방법을 보자.

```java
	@Bean
	@StepScope
	public JsonItemReader<Customer> customerFileReader(
		@Value("#{jobParameters['customerFile']}") Resource inputFile) {

		// objectMapper 날짜 형식 커스텀하기 위해서 직접 작성
		ObjectMapper objectMapper = new ObjectMapper();
		objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd hh:mm:ss"));

		JacksonJsonObjectReader<Customer> jsonObjectReader = new JacksonJsonObjectReader<>(Customer.class);
		jsonObjectReader.setMapper(objectMapper);

		return new JsonItemReaderBuilder<Customer>()
				.name("customerFileReader")	// 배치 재시작 시 사용할 배치 이름
				.jsonObjectReader(jsonObjectReader)	// 파싱에 필요한 jsonObjectReader
				.resource(inputFile)	// 읽어들일 resource
				.build();
	}
```

## 데이터베이스
### JDBC
대용량 데이터를 처리할 때 한번에 데이터를 읽어와 객체를 생성하는 일을 방지하기 위해서 `cursor`와 `paging` 기법을 제공한다.

- cursor
Set으로 구현
ResultSet이 open되면 next() 메서드를 호출할 때마다 데이터베이스에서 레코드 가져와 반환

- paging
페이지(청크) 크기만큼 레코드 가져오기

- cursor 기반 jdbc 코드

```java
	@Bean
	public JdbcCursorItemReader<Customer> customerItemReader(DataSource dataSource) {
		return new JdbcCursorItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.dataSource(dataSource)
				.sql("select * from customer where city = ?")
				.rowMapper(new CustomerRowMapper())	// 쿼리 결과를 도메인 객체로 매핑하는 매퍼
				.preparedStatementSetter(citySetter(null))
				.build();
	}
    
    	@Bean
	@StepScope
    	// 스프링에서 제공하는 ArgumentPreparedStatementSetter 이용해서 argument 넘기기
	public ArgumentPreparedStatementSetter citySetter(
			@Value("#{jobParameters['city']}") String city) {

		return new ArgumentPreparedStatementSetter(new Object [] {city});
	}
```

하지만 cursor 기법을 사용할 경우 대용량 데이터를 처리할 때 매번 요청마다 네트워크 오버헤드가 추가되며 ResultSet은 non-thread safe하므로 멀티 스레드 환경에서 사용할 수 없는 단점이 있다.

- 페이징 기반 JDBC 코드
잡이 처리할 아이템은 한건씩 처리된다는 점에서 cursor 기반과 동일하지만 DB에서 레코드를 가져올 때 차이가 난다. 커서 기법은 한번에 SQL 쿼리 하나를 실행해 레코드를 하나씩 가져오지만 페이징 기법은 각 페이지마다 새로운 쿼리를 실행한 뒤 쿼리 결과를 한번에 메모리에 적재한다.

페이징 기반에는 데이터베이스에서 제공하는 페이징 구현체를 사용하는 방법도 있지만 스프링에서 제공하는 sqlPagingQueryProviderFactoryBean을 사용하면 데이터베이스를 감지하여 알맞은 페이징 구현체를 반환한다.

```java
@Bean
	@StepScope
	public JdbcPagingItemReader<Customer> customerItemReader(DataSource dataSource,
			PagingQueryProvider queryProvider,
			@Value("#{jobParameters['city']}") String city) {

		Map<String, Object> parameterValues = new HashMap<>(1);
		parameterValues.put("city", city);

		return new JdbcPagingItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.dataSource(dataSource)
				.queryProvider(queryProvider)
				.parameterValues(parameterValues)
				.pageSize(10)
				.rowMapper(new CustomerRowMapper())
				.build();
	}

	@Bean
	public SqlPagingQueryProviderFactoryBean pagingQueryProvider(DataSource dataSource) {
		SqlPagingQueryProviderFactoryBean factoryBean = new SqlPagingQueryProviderFactoryBean();

		factoryBean.setDataSource(dataSource);
		factoryBean.setSelectClause("select *");
		factoryBean.setFromClause("from Customer");
        // ? 대신 named parameter 이용
		factoryBean.setWhereClause("where city = :city");
        // ResultSet에서 중복 금지
		factoryBean.setSortKey("lastName");

		return factoryBean;
	}
```

### 하이버네이트
하이버네이트의 기본 세션은 stateful하기 때문에 대용량 데이터를 읽을 경우 해당 데이터를 캐시에 쌓게 되고 OOM이 일어날 수 있다. 또한 직접 JDBC를 사용할 때보다 더 큰 부하를 유발한다. 이를 해결하기 위해서 하이버네이트 기반 ItemReader를 사용한다.
하이버네이트 기반 ItemReader는 커밋할 때 세션을 flush하여 캐시 문제를 해결했고 매핑 기능이 막강하다.

먼저 BatchConfigurer을 사용해서 HibernateTransactionManager을 구성한다.

- 하이버네이트 커서

```java
	@Bean
	@StepScope
	public HibernateCursorItemReader<Customer> customerItemReader(
			EntityManagerFactory entityManagerFactory,
			@Value("#{jobParameters['city']}") String city) {

		return new HibernateCursorItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
				.queryString("from Customer where city = :city")
				.parameterValues(Collections.singletonMap("city", city))
				.build();
	}
```

- 하이버네이트 페이징

```java
	@Bean
	@StepScope
	public HibernatePagingItemReader<Customer> customerItemReader(
			EntityManagerFactory entityManagerFactory,
			@Value("#{jobParameters['city']}") String city) {

		return new HibernatePagingItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
				.queryString("from Customer where city = :city")
				.parameterValues(Collections.singletonMap("city", city))
				.pageSize(10)
				.build();
	}
```

### JPA

JPA는 커서 기반 reader가 존재하지 않고 페이징 기반 reader만 존재한다.
JPA를 사용하면 BatchConfigurer 구현체를 생성할 필요 없이 SpringBoot가 JpaTransactionManager 구성을 한다. 그래서 개발자가 유일하게 신경써야 할 부분은 ItemReader를 구성하는 것이다. 

```java
	@Bean
	@StepScope
	public JpaPagingItemReader<Customer> customerItemReader(
			EntityManagerFactory entityManagerFactory,
			@Value("#{jobParameters['city']}") String city) {

		CustomerByCityQueryProvider queryProvider =
				new CustomerByCityQueryProvider();
		queryProvider.setCityName(city);

		return new JpaPagingItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.entityManagerFactory(entityManagerFactory)
				.queryProvider(queryProvider)
				.parameterValues(Collections.singletonMap("city", city))
				.build();
	}
```

### 저장 프로시저

저장 프로시저란 데이터 베이스 전용 코드의 집합이다. 스프링 배치가 저장 프로시저에서 데이터를 조회할 때는 StoredProcedureItemReader를 사용한다.

MySQL 예시를 살펴보자.

```java
@Bean
	@StepScope
	public StoredProcedureItemReader<Customer> customerItemReader(DataSource dataSource,
			@Value("#{jobParameters['city']}") String city) {

		return new StoredProcedureItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.dataSource(dataSource)
				.procedureName("customer_list")
				.parameters(new SqlParameter[]{new SqlParameter("cityOption", Types.VARCHAR)})
				.preparedStatementSetter(new ArgumentPreparedStatementSetter(new Object[] {city}))
				.rowMapper(new CustomerRowMapper())
				.build();
	}
```

### 스프링 데이터
스프링 데이터는 단일 프로젝트가 아니며 여러 프로젝트의 포트폴리오다. 각 프로젝트는 추상화된 집합(Repository 인터페이스)를 제공한다. NoSQL 데이터 저장소에 저장된 데이터를 사용하는 방법을 살펴보자.

- 몽고 DB

```java
	@Bean
	@StepScope
	public MongoItemReader<Map> tweetsItemReader(MongoOperations mongoTemplate,
			@Value("#{jobParameters['hashTag']}") String hashtag) {
		return new MongoItemReaderBuilder<Map>()
				.name("tweetsItemReader")	// 잡 재시작할 수 있도록 ExecutionContext에 상태 저장하는데 사용
				.targetType(Map.class)	// 리턴 문서 역직렬화할 클래스
				.jsonQuery("{ \"entities.hashtags.text\": { $eq: ?0 }}")	// 쿼리
				.collection("tweets_collection")	// 쿼리 컬렉션
				.parameterValues(Collections.singletonList(hashtag))
				.pageSize(10)
				.sorts(Collections.singletonMap("created_at", Sort.Direction.ASC))
				.template(mongoTemplate)	// 쿼리 실행 대상 MongoOprtations 구현체
				.build();
	}
```

- 스프링 데이터 repository

스프링 데이터 repository란 스프링 데이터가 제공하는 특정 인터페이스 중 하나를 상속하는 인터페이스로 이를 사용자가 정의하면 해당 인터페이스의 구현을 처리하는 기능을 제공한다.
스프링 배치가 스프링 데이터와 호환성이 좋은 이유는 스프링 배치가 스프링 데이터의 PagingAndSortingRepository를 활용하기 때문이다.

```java
	@Bean
	@StepScope
	public RepositoryItemReader<Customer> customerItemReader(CustomerRepository repository,
			@Value("#{jobParameters['city']}") String city) {

		return new RepositoryItemReaderBuilder<Customer>()
				.name("customerItemReader")
				.arguments(Collections.singletonList(city))
                // 스프링 데이터 repository에 정의된 메서드
				.methodName("findByCity")
				.repository(repository)
				.sorts(Collections.singletonMap("lastName", Sort.Direction.ASC))
				.build();
	}
```

## 기존 서비스

기존 서비스에서 사용하던 코드를 이용해서 데이터를 읽으려면 ItemReaderAdapter를 사용한다. 이 때 두가지를 주의하자.

1. ItemReader가 반환하는 객체가 컬렉션이면 개발자는 직접 객체를 하나씩 꺼내서 처리해야 한다.
2. 입력 데이터를 모두 처리하면 서비스 메서드는 반드시 null을 반환해서 해당 스텝의 입력을 모두 소비했음을 알려야 한다.

## 커스텀 입력
잡의 여러 상태에 따라서 어떻게 입력을 할지 reader를 커스텀할 수 있다. 이 때 reader의 상태를 저장해서 이전의 종료된 지점부터 다시 시작하기를 원한다면 ItemStream 인터페이스를 구현해야 한다.

[https://github.com/Apress/def-guide-spring-batch/blob/master/Chapter07/src/main/java/com/example/Chapter07/batch/CustomerItemReader.java](코드 url)

## 에러 처리

### 레코드 건너뛰기
에러가 발생했을 때 레코드를 건너뛰는 방법이 있다. 이 때 두가지를 고려해야 한다.
1. 어떤 조건에서 건너뛸 것인가
2. 얼마나 많은 레코드를 건너뛸 것인가

```java
@Bean
public Step copyFileStep() {
	return this.stepBuilderFactory.get("copyFileStep")
    	.<Customer, Customer>chunk(10)
        .reader(customerItemReader())
        .writer(itemWriter())
        .faultTolerant()
        // ParseException 예외를 제외한 예외를 10번까지 스킵한다.
        .skip(Exception.class)
        .noSkip(ParseException.class)
        .skipLimit(10)
        .build();
}
```

skipPolicy라는 스프링 배치에서 제공하는 인터페이스를 이용해서 이를 따로 분리할 수 있다.

```java
public class FileVerificationSkipper implements SkipPolicy {
	public boolean shouldSkip(Throwable exception, int skipCount) {
    	throws SkipLimitExceededException {
        	if (exception instanceof FileNotFoundException)
            	return false;
            else if (exception instanceof ParseException && skipCount <= 10)
            	return true;
            else
            	return false;
        }
    
    }
}
```

### 잘못된 레코드 로그 남기기

돈과 관련된 배치라면 레코드를 뛰어넘는 것은 제대로 된 해결책이 아니다. 이 때 ItemListener를 이용해서 로그를 남긴다.

```java
public class CustomerItemListener  {

	private static final Log logger = LogFactory.getLog(CustomerItemListener.class);

	// ItemListenerSupport의 onReadError 메서드를 오버라이드하거나 POJO에 어노테이션 달아서 사용
	@OnReadError
	public void onReadError(Exception e) {
		if(e instanceof FlatFileParseException) {
			FlatFileParseException ffpe = (FlatFileParseException) e;

			StringBuilder errorMessage = new StringBuilder();
			errorMessage.append("An error occured while processing the " +
					ffpe.getLineNumber() +
					" line of the file.  Below was the faulty " +
					"input.\n");
			errorMessage.append(ffpe.getInput() + "\n");

			logger.error(errorMessage.toString(), ffpe);
		} else {
			logger.error("An error has occurred", e);
		}
	}
}
```

### 입력이 없을 때의 처리
데이터가 비어있는 입력 소스를 읽은 경우는 스프링 배치에서 일반적인 경우는 아니다. 해당 경우를 처리하고 싶을 때는 StepListener를 사용한다.

```java
public class EmptyInputStepFailer {
	@AfterStep
    public ExitStatus afterStep(StepExecution execution) {
    	if (execution.getReadCount() > 0)
        	return execution.getExitStatus();
        else
        	return ExitStatus.FAILED;
    }
}
```
