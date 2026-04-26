# 08 — Spring Batch

## 1. Architecture Spring Batch

```
Job
└── Step 1 : Lire le CSV
    ├── ItemReader   (lit N items)
    ├── ItemProcessor (transforme chaque item)
    └── ItemWriter   (écrit par chunks)
└── Step 2 : Générer rapport
└── Step 3 : Envoyer email
```

### Modèle de données Spring Batch

| Table | Rôle |
|-------|------|
| `BATCH_JOB_INSTANCE` | Une exécution unique d'un job (par paramètres) |
| `BATCH_JOB_EXECUTION` | Exécution concrète (peut être relancée) |
| `BATCH_STEP_EXECUTION` | Stats par step (read/write/skip count) |
| `BATCH_JOB_EXECUTION_PARAMS` | Paramètres de l'exécution |

---

## 2. Configuration d'un Job

```java
@Configuration
@EnableBatchProcessing
public class ImportOrdersJobConfig {

    @Bean
    public Job importOrdersJob(JobRepository jobRepository, Step readAndProcessStep, Step reportStep) {
        return new JobBuilder("importOrdersJob", jobRepository)
            .start(readAndProcessStep)
            .next(reportStep)
            .incrementer(new RunIdIncrementer())  // permet re-exécution
            .listener(new JobCompletionListener())
            .build();
    }

    @Bean
    public Step readAndProcessStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        ItemReader<OrderCsvLine> reader,
        ItemProcessor<OrderCsvLine, Order> processor,
        ItemWriter<Order> writer
    ) {
        return new StepBuilder("readAndProcessStep", jobRepository)
            .<OrderCsvLine, Order>chunk(100, transactionManager) // chunk de 100
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skipLimit(10)
            .skip(InvalidOrderException.class)     // sauter les lignes invalides
            .retryLimit(3)
            .retry(TransientDataAccessException.class)  // retry les erreurs DB transitoires
            .listener(new ChunkExecutionListener())
            .build();
    }
}
```

---

## 3. Reader, Processor, Writer

```java
// FlatFileItemReader : lecture CSV
@Bean
@StepScope
public FlatFileItemReader<OrderCsvLine> csvReader(
    @Value("#{jobParameters['inputFile']}") String inputFile
) {
    return new FlatFileItemReaderBuilder<OrderCsvLine>()
        .name("orderCsvReader")
        .resource(new FileSystemResource(inputFile))
        .linesToSkip(1)  // header
        .delimited()
        .delimiter(";")
        .names("reference", "customerId", "amount", "date")
        .fieldSetMapper(new BeanWrapperFieldSetMapper<>() {{
            setTargetType(OrderCsvLine.class);
        }})
        .build();
}

// JdbcCursorItemReader : lecture BDD
@Bean
@StepScope
public JdbcCursorItemReader<Order> dbReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Order>()
        .name("orderDbReader")
        .dataSource(dataSource)
        .sql("SELECT * FROM orders WHERE status = 'PENDING' AND created_at < ?")
        .preparedStatementSetter(ps -> ps.setDate(1, Date.valueOf(LocalDate.now().minusDays(7))))
        .rowMapper(new OrderRowMapper())
        .build();
}

// ItemProcessor
@Component
public class OrderProcessor implements ItemProcessor<OrderCsvLine, Order> {

    @Override
    public Order process(OrderCsvLine line) throws Exception {
        // Retourner null = filtrer (ne pas écrire) cet item
        if (line.getAmount() <= 0) {
            log.warn("Filtering invalid order with amount {}", line.getAmount());
            return null;
        }

        Customer customer = customerRepo.findById(line.getCustomerId())
            .orElseThrow(() -> new InvalidOrderException("Unknown customer: " + line.getCustomerId()));

        return Order.builder()
            .reference(line.getReference())
            .customer(customer)
            .amount(new BigDecimal(line.getAmount()))
            .status(OrderStatus.IMPORTED)
            .build();
    }
}

// JdbcBatchItemWriter : écriture BDD en batch (efficace)
@Bean
public JdbcBatchItemWriter<Order> dbWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Order>()
        .sql("INSERT INTO orders (reference, customer_id, amount, status) VALUES (:reference, :customerId, :amount, :status)")
        .dataSource(dataSource)
        .beanMapped()  // utilise les getters de l'objet
        .build();
}
```

---

## 4. Scheduling & Lancement

```java
// Lancement programmatique
@Service
public class BatchLauncher {

    private final JobLauncher jobLauncher;
    private final Job importOrdersJob;

    @Scheduled(cron = "0 0 2 * * *")  // tous les jours à 2h
    public void launchNightly() throws JobExecutionException {
        JobParameters params = new JobParametersBuilder()
            .addString("inputFile", "/data/orders-" + LocalDate.now() + ".csv")
            .addLong("timestamp", System.currentTimeMillis())  // garantit unicité
            .toJobParameters();

        JobExecution execution = jobLauncher.run(importOrdersJob, params);
        log.info("Job {} finished with status: {}", execution.getJobId(), execution.getStatus());
    }
}

// Listener pour monitoring
@Component
public class JobCompletionListener implements JobExecutionListener {

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            log.info("Job completed: {} items processed", jobExecution
                .getStepExecutions().stream()
                .mapToLong(StepExecution::getWriteCount).sum());
        } else if (jobExecution.getStatus() == BatchStatus.FAILED) {
            alertService.sendAlert("Batch job failed", jobExecution.getAllFailureExceptions());
        }
    }
}
```

---

## Questions d'Interview — Spring Batch

---

### Q1 : Qu'est-ce que le chunk processing et pourquoi l'utiliser ?

**Réponse :**

Le chunk processing lit N items, les traite, puis les écrit en une seule transaction. Si l'écriture échoue, seul le chunk courant est rollbacké.

```
Read   : [item1, item2, ..., item100]  ← lecture sans transaction
Process: [item1→order1, item2→order2, ...]
Write  : INSERT 100 rows               ← 1 seule transaction, 1 batch SQL

Vs ligne par ligne :
- 1 million de lignes = 1 million de transactions = très lent
- Chunk de 1000 = 1000 transactions
```

**Taille du chunk :** compromis entre mémoire (grand chunk = plus de RAM) et performances (petit chunk = plus de commits).

---

### Q2 : Comment relancer un job en cas d'échec à partir de là où il s'est arrêté ?

**Réponse :**

Spring Batch maintient l'état d'exécution dans ses tables de métadonnées. Si un job échoue, le relancer avec les **mêmes paramètres** reprend là où il s'est arrêté.

```java
// Relance automatique (même JobParameters)
jobLauncher.run(importJob, sameParameters);
// Spring Batch reprend depuis le dernier checkpoint

// Pour forcer une nouvelle instance (ignorer l'historique)
new JobParametersBuilder()
    .addLong("run.id", System.currentTimeMillis())  // nouveau paramètre = nouvelle instance
    .toJobParameters();
```

**Configuration :**
```java
.step("processStep")
    .allowStartIfComplete(false)  // default : skip si déjà COMPLETED
```

---

### Q3 : Comment gérer les erreurs de processing (skip, retry) ?

**Réponse :**

```java
.<CsvLine, Order>chunk(100)
    .faultTolerant()

    // Skip : ignorer les items problématiques
    .skipLimit(50)                              // max 50 items sautés
    .skip(InvalidDataException.class)           // type d'exception à sauter
    .noSkip(CriticalException.class)            // ne jamais sauter ce type

    // Retry : retenter les items en erreur
    .retryLimit(3)
    .retry(TransientDataAccessException.class)  // erreurs DB transitoires

    // Listener pour loguer les skips
    .listener(new SkipListener<CsvLine, Order>() {
        @Override
        public void onSkipInProcess(CsvLine item, Throwable t) {
            log.warn("Skipped item: {} due to {}", item, t.getMessage());
            skipRepo.save(new SkippedItem(item, t.getMessage()));
        }
    });
```

---

## Pièges Classiques

- Utiliser `@Autowired` dans un `@StepScope` bean sans `@StepScope` annotation → bean partagé entre steps
- Lire et écrire dans la même table sans pagination correcte → curseur infini
- Chunk trop grand → `OutOfMemoryError` si les items sont gros
- Ne pas sauvegarder le contexte step → impossible de reprendre après crash
- `JobParametersIncrementer` non configuré → `JobInstanceAlreadyCompleteException` au second lancement
