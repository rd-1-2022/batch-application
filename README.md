== Spring Batch Application

The following steps were used to create this Spring Batch Application that will read Person data from a CSV file, transform the data, and then store it in a relational database.  We will use the H2 database in this example.

1. Add the following dependeicnes to the pom.xml file

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-batch</artifactId>
		</dependency>

		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.batch</groupId>
			<artifactId>spring-batch-test</artifactId>
			<scope>test</scope>
		</dependency>
```

2.  Create a simple Person domain object

```java
package com.example.batchprocessing;

public class Person {

  private String lastName;
  private String firstName;

  public Person() {
  }

  public Person(String firstName, String lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  public void setFirstName(String firstName) {
    this.firstName = firstName;
  }

  public String getFirstName() {
    return firstName;
  }

  public String getLastName() {
    return lastName;
  }

  public void setLastName(String lastName) {
    this.lastName = lastName;
  }

  @Override
  public String toString() {
    return "firstName: " + firstName + ", lastName: " + lastName;
  }

}
```

3. Create some sample data in a CSV file

```csv
# filename: src/main/resources/sample-data.csv
Jill,Doe
Joe,Doe
Justin,Doe
Jane,Doe
John,Doe
```

4. Create a database schema to support storing the Person object.

```sql
-- filename: src/main/resources/schema-all.sql:
DROP TABLE people IF EXISTS;

CREATE TABLE people  (
    person_id BIGINT IDENTITY NOT NULL PRIMARY KEY,
    first_name VARCHAR(20),
    last_name VARCHAR(20)
);
```

**NOTE**
Spring Boot runs schema-@@platform@@.sql automatically during startup. -all is the default for all platforms.

5. Create an `ItemProcessor` to transform the CSV data.  In this case we will convert the firstname and lastname to uppercase.

```java
package com.example.batchprocessing;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import org.springframework.batch.item.ItemProcessor;

public class PersonItemProcessor implements ItemProcessor<Person, Person> {

	private static final Logger log = LoggerFactory.getLogger(PersonItemProcessor.class);

	@Override
	public Person process(final Person person) throws Exception {
		final String firstName = person.getFirstName().toUpperCase();
		final String lastName = person.getLastName().toUpperCase();

		final Person transformedPerson = new Person(firstName, lastName);

		log.info("Converting (" + person + ") into (" + transformedPerson + ")");

		return transformedPerson;
	}

}
```

`PersonItemProcessor` implements Spring Batch’s `ItemProcessor` interface. This makes it easy to wire the code into a batch job that you will define later in this guide. According to the interface, you receive an incoming `Person` object, after which you transform it to an upper-cased `Person`.

**NOTE**
The input and output types need not be the same. In fact, after one source of data is read, sometimes the application’s data flow needs a different data type.

6. Put together a Batch Job

To configure your job, you must first create a Spring @Configuration class This example uses a memory-based database, meaning that, when it is done, the data is gone. The following `BatchConfiguration` class defines a reader, a processor, and a writer.

```java
package com.example.batchprocessing;

import javax.sql.DataSource;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.database.BeanPropertyItemSqlParameterSourceProvider;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class BatchConfiguration {

	@Bean
	public FlatFileItemReader<Person> reader() {
		return new FlatFileItemReaderBuilder<Person>()
				.name("personItemReader")
				.resource(new ClassPathResource("sample-data.csv"))
				.delimited()
				.names(new String[]{"firstName", "lastName"})
				.fieldSetMapper(new BeanWrapperFieldSetMapper<Person>() {{
					setTargetType(Person.class);
				}})
				.build();
	}

	@Bean
	public PersonItemProcessor processor() {
		return new PersonItemProcessor();
	}

	@Bean
	public JdbcBatchItemWriter<Person> writer(DataSource dataSource) {
		return new JdbcBatchItemWriterBuilder<Person>()
				.itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
				.sql("INSERT INTO people (first_name, last_name) VALUES (:firstName, :lastName)")
				.dataSource(dataSource)
				.build();
	}

	@Bean
	public Step myBatchStep(JobRepository jobRepository,
			PlatformTransactionManager transactionManager, JdbcBatchItemWriter<Person> writer) {
		return new StepBuilder("step1", jobRepository)
				.<Person, Person> chunk(10, transactionManager)
				.reader(reader())
				.processor(processor())
				.writer(writer)
				.build();
	}

	@Bean
	public Job myBatchJob(JobRepository jobRepository,
			JobCompletionNotificationListener listener, Step step1) {
		return new JobBuilder("importUserJob", jobRepository)
				.incrementer(new RunIdIncrementer())
				.listener(listener)
				.flow(step1)
				.end()
				.build();
	}
	
}
```

Jobs are built from steps, where each step can involve a reader, a processor, and a writer.

The first three `@Bean` definitions create the `reader`, `processor`, and `writer`.  These are then joined together into in a `Step`, and then the `Step` is included in a Batch `Job`.

* `reader` creates an `ItemReader`. It looks for a file called `sample-data.csv` and parses each line item with enough information to turn it into a Person.

* `processor` creates a `PersonItemProcessor`, which converts the data to upper case.

* `writer` creates an `ItemWriter`. This one is aimed at a JDBC destination and automatically gets a copy of the dataSource created by `@EnableBatchProcessing`. It includes the SQL statement needed to insert a single Person, driven by Java bean properties.

* `myBatchStep` defines how much data to write at a time. In this case, it writes up to ten records at a time. Note, that the method `chunk()` is prefixed <Person,Person> because it is a generic method. This represents the input and output types of each "chunk" of processing and lines up with ItemReader<Person> and ItemWriter<Person>.

* `myBatchJob`  defines the Job.  You need an incrementer, because jobs use a database to maintain execution state. You then add the step to the job.  

The `JobCompletionNotificationListener` is defined as a Spring `@Component` and is used in the `myBatchJob` @Bean definition.  The `JobCompletionNotificationListener` is a simple way to see a notification messages when the job completes.

7.  Create the `JobCompletionNotificationListener` class

```java
package com.example.batchprocessing;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.core.BatchStatus;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class JobCompletionNotificationListener implements JobExecutionListener {

	private static final Logger log = LoggerFactory.getLogger(JobCompletionNotificationListener.class);

	private final JdbcTemplate jdbcTemplate;

	@Autowired
	public JobCompletionNotificationListener(JdbcTemplate jdbcTemplate) {
		this.jdbcTemplate = jdbcTemplate;
	}

	@Override
	public void afterJob(JobExecution jobExecution) {
		if(jobExecution.getStatus() == BatchStatus.COMPLETED) {
			log.info("!!! JOB FINISHED! Time to verify the results");

			jdbcTemplate.query("SELECT first_name, last_name FROM people",
					(rs, row) -> new Person(
							rs.getString(1),
							rs.getString(2))
			).forEach(person -> log.info("Found <{{}}> in the database.", person));
		}
	}
} 
```

8.  Create the main Spring Application class to run the code

```java 
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

	public static void main(String[] args) throws Exception {
		SpringApplication.run(Application.class, args);
	}
}
```

9. Run the application

```bash
./mvnw spring-boot:run
```

You should see output similar to the following


```bash
Converting (firstName: Jill, lastName: Doe) into (firstName: JILL, lastName: DOE)
Converting (firstName: Joe, lastName: Doe) into (firstName: JOE, lastName: DOE)
Converting (firstName: Justin, lastName: Doe) into (firstName: JUSTIN, lastName: DOE)
Converting (firstName: Jane, lastName: Doe) into (firstName: JANE, lastName: DOE)
Converting (firstName: John, lastName: Doe) into (firstName: JOHN, lastName: DOE)
Found <firstName: JILL, lastName: DOE> in the database.
Found <firstName: JOE, lastName: DOE> in the database.
Found <firstName: JUSTIN, lastName: DOE> in the database.
Found <firstName: JANE, lastName: DOE> in the database.
Found <firstName: JOHN, lastName: DOE> in the database.
```
