# projectdetails

1. whenever a project given with json data then to parse and load on to database
first create an entity 
here in entity it should be based on what we are giving in the sql table 
sample entity as follows

```
@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table(name = "receipe")
public class Recipe {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String cuisine;
    private String title;
    private Float rating;


    private Integer prepTime;
    private Integer cookTime;
    private Integer totalTime;
```
create the dto 
here dto means it is data transfer object here our json values and entity values are different hence to match json structure and here ignore case is utlilised to ignore unwanted fields 
```
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class RecipeDTO {
    private String cuisine;
    private String title;
    private Object rating;
    private Object prep_time;
    private Object cook_time;
    private Object total_time;
    @Column(columnDefinition = "LONGTEXT")
    private String description;
    @Column(columnDefinition = "LONGTEXT")
    private Map<String,Object> nutrients;
    private Object serves;

}
```
create the repository 
create the repository with the help of jparepository and japspecificationExecutor 
```
@Repository
public interface RecipeRepository extends JpaRepository<Recipe, Long>, JpaSpecificationExecutor<Recipe> {

}
```
i will give the service logic for json loading automatically here also given pagination concept and sorting concept 
```
@Service
@RequiredArgsConstructor
public class RecipeService {

    private final RecipeRepository recipeRepository;
    private ObjectMapper objectMapper = new ObjectMapper();

    // 🔥 Load JSON Automatically at Startup
    @PostConstruct
    public void loadJsonData() throws Exception {
        if (recipeRepository.count() > 0) {
            return;
        }

        File file = new ClassPathResource("US_recipes_null.json").getFile();

        Map<String, RecipeDTO> data =
                objectMapper.readValue(file,
                        new TypeReference<Map<String, RecipeDTO>>() {});

        for (RecipeDTO dto : data.values()) {

            Recipe recipe = new Recipe();

            recipe.setCuisine(dto.getCuisine());
            recipe.setTitle(dto.getTitle());
            recipe.setRating(parseFloat(dto.getRating()));
            recipe.setPrepTime(parseInt(dto.getPrep_time()));
            recipe.setCookTime(parseInt(dto.getCook_time()));
            recipe.setTotalTime(parseInt(dto.getTotal_time()));
            recipe.setDescription(dto.getDescription());
            recipe.setNutrients(objectMapper.writeValueAsString(dto.getNutrients()));
            recipe.setServes(parseString(dto.getServes()));

            recipeRepository.save(recipe);
        }
    }

    // 🔥 Pagination + Sorting
    public Page<Recipe> getAllRecipes(int page, int limit) {

        Pageable pageable = PageRequest.of(
                page - 1,
                limit,
                Sort.by("rating").descending()
        );

        return recipeRepository.findAll(pageable);
    }

    // 🔥 Dynamic Search
    public List<Recipe> searchRecipes(
            String title,
            String cuisine,
            Float rating,
            Integer totalTime) {

        Specification<Recipe> spec = (root, query, cb) -> cb.conjunction();

        if (title != null) {
            spec = spec.and((root, query, cb) ->
                    cb.like(cb.lower(root.get("title")),
                            "%" + title.toLowerCase() + "%"));
        }

        if (cuisine != null) {
            spec = spec.and((root, query, cb) ->
                    cb.equal(root.get("cuisine"), cuisine));
        }

        if (rating != null) {
            spec = spec.and((root, query, cb) ->
                    cb.greaterThanOrEqualTo(root.get("rating"), rating));
        }

        if (totalTime != null) {
            spec = spec.and((root, query, cb) ->
                    cb.lessThanOrEqualTo(root.get("totalTime"), totalTime));
        }

        return recipeRepository.findAll(spec);
    }

    // 🔥 Handle NaN
    private Float parseFloat(Object value) {
        if (value == null) return null;
        if (value.toString().equalsIgnoreCase("NaN")) return null;
        return Float.parseFloat(value.toString());
    }

    private Integer parseInt(Object value) {
        if (value == null) return null;
        if (value.toString().equalsIgnoreCase("NaN")) return null;
        return Integer.parseInt(value.toString());
    }

    private String parseString(Object value) {
        if (value == null) return null;
        if (value.toString().equalsIgnoreCase("NaN")) return null;
        return value.toString();
    }
}

```

this is rest controller
```
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class RecipeController {

    private final RecipeService recipeService;

    // API 1
    @GetMapping("/recipes")
    public Page<Recipe> getAllRecipes(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int limit) {

        return recipeService.getAllRecipes(page, limit);
    }

    // API 2
    @GetMapping("/search")
    public List<Recipe> searchRecipes(
            @RequestParam(required = false) String title,
            @RequestParam(required = false) String cuisine,
            @RequestParam(required = false) Float rating,
            @RequestParam(required = false) Integer totalTime) {

        return recipeService.searchRecipes(title, cuisine, rating, totalTime);
    }
}
```


2. if a project has been given a csv file or excel then to load that data into database then to find given options is that 
refer my github to get 


3. If and api was given use RestTemplate u should create this as a particular class file
```
@Service
public class ApiService {

    @Autowired
    private RestTemplate restTemplate;

    public ExternalResponse fetchData() {
        String url = "https://api.example.com/data";
        return restTemplate.getForObject(url, ExternalResponse.class);
    }
}

```

This is were u will fetch the api data to db 
```
@Service
public class WeatherService {

    @Autowired
    private ApiService apiService;

    @Autowired
    private WeatherRepository weatherRepository;

    public void fetchAndStore() {

        ExternalResponse response = apiService.fetchData();

        WeatherData data = new WeatherData();
        data.setTemperature(response.getTemp());
        data.setHumidity(response.getHumidity());
        data.setTimestamp(LocalDateTime.now());

        weatherRepository.save(data);
    }
}
```


ok lets see for an api given how 
com.example.cve

├── CveApplication.java
│
├── config
│   └── AppConfig.java
│
├── controller
│   └── CveController.java
│
├── service
│   ├── CveService.java
│   └── NvdApiService.java
│
├── repository
│   └── CveRepository.java
│
├── entity
│   └── Cve.java
│
├── dto
│   ├── CveResponseDto.java
│   └── NvdResponseDto.java
│
├── scheduler
│   └── CveScheduler.java
│
├── exception
│   └── GlobalExceptionHandler.java

config 
```
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}


```

cretate an entity
```
@Entity
@Table(name = "cve")
public class Cve {

    @Id
    private String cveId;

    @Column(length = 5000)
    private String description;

    private Double baseScore;

    private String severity;

    private LocalDate publishedDate;

    private LocalDate lastModifiedDate;

    private Integer year;

    // getters & setters
}
```
repository
```
@Repository
public interface CveRepository extends JpaRepository<Cve, String> {

    Page<Cve> findByYear(Integer year, Pageable pageable);

    Page<Cve> findByBaseScoreGreaterThanEqual(Double score, Pageable pageable);

    Page<Cve> findByLastModifiedDateAfter(LocalDate date, Pageable pageable);
}
```
service layer of nvd api service layer where it fetch and saves it in db

```
@Service
public class NvdApiService {

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private CveRepository cveRepository;

    private final String BASE_URL =
            "https://services.nvd.nist.gov/rest/json/cves/2.0";

    public void fetchAndStoreCves() {

        int startIndex = 0;
        int resultsPerPage = 2000;
        boolean hasMore = true;

        while (hasMore) {

            String url = BASE_URL + "?startIndex=" + startIndex +
                    "&resultsPerPage=" + resultsPerPage;

            ResponseEntity<NvdResponseDto> response =
                    restTemplate.getForEntity(url, NvdResponseDto.class);

            List<NvdResponseDto.Vulnerability> list =
                    response.getBody().getVulnerabilities();

            if (list == null || list.isEmpty()) {
                hasMore = false;
            } else {

                for (NvdResponseDto.Vulnerability v : list) {

                    String cveId = v.getCve().getId();

                    Cve entity = new Cve();
                    entity.setCveId(cveId);
                    entity.setDescription(v.getCve().getDescriptions().get(0).getValue());
                    entity.setPublishedDate(
                        LocalDate.parse(v.getCve().getPublished().substring(0,10))
                    );
                    entity.setLastModifiedDate(
                        LocalDate.parse(v.getCve().getLastModified().substring(0,10))
                    );
                    entity.setYear(Integer.parseInt(cveId.substring(4,8)));

                    // Extract score (v3 preferred)
                    Double score = extractScore(v);
                    entity.setBaseScore(score);

                    cveRepository.save(entity);
                }

                startIndex += resultsPerPage;
            }
        }
    }

    private Double extractScore(NvdResponseDto.Vulnerability v) {
        try {
            return v.getCve()
                    .getMetrics()
                    .getCvssMetricV31()
                    .get(0)
                    .getCvssData()
                    .getBaseScore();
        } catch (Exception e) {
            return null;
        }
    }
}
```
Businness service layer where u logic to implememt will be given 

```
@Service
public class CveService {

    @Autowired
    private CveRepository cveRepository;

    public Page<Cve> getAll(Pageable pageable) {
        return cveRepository.findAll(pageable);
    }

    public Cve getById(String id) {
        return cveRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("CVE Not Found"));
    }

    public Page<Cve> getByYear(Integer year, Pageable pageable) {
        return cveRepository.findByYear(year, pageable);
    }

    public Page<Cve> getByScore(Double score, Pageable pageable) {
        return cveRepository.findByBaseScoreGreaterThanEqual(score, pageable);
    }

    public Page<Cve> getLastModified(int days, Pageable pageable) {
        LocalDate date = LocalDate.now().minusDays(days);
        return cveRepository.findByLastModifiedDateAfter(date, pageable);
    }
}
```

controller where u will return all the things to api 

```
@RestController
@RequestMapping("/api/cves")
public class CveController {

    @Autowired
    private CveService cveService;

    @GetMapping("/list")
    public Page<Cve> list(Pageable pageable) {
        return cveService.getAll(pageable);
    }

    @GetMapping("/{id}")
    public Cve getById(@PathVariable String id) {
        return cveService.getById(id);
    }

    @GetMapping("/year/{year}")
    public Page<Cve> getByYear(@PathVariable Integer year,
                               Pageable pageable) {
        return cveService.getByYear(year, pageable);
    }

    @GetMapping("/score")
    public Page<Cve> getByScore(@RequestParam Double min,
                                Pageable pageable) {
        return cveService.getByScore(min, pageable);
    }

    @GetMapping("/last-modified")
    public Page<Cve> lastModified(@RequestParam int days,
                                  Pageable pageable) {
        return cveService.getLastModified(days, pageable);
    }
}
```

here Schedular
```
@Component
public class CveScheduler {

    @Autowired
    private NvdApiService nvdApiService;

    @Scheduled(cron = "0 0 2 * * ?") // Everyday 2AM
    public void syncData() {
        nvdApiService.fetchAndStoreCves();
    }
```
apllication.properties
```
spring.datasource.url=jdbc:mysql://localhost:3306/cvedb
spring.datasource.username=root
spring.datasource.password=root

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

spring.jackson.deserialization.fail-on-unknown-properties=false
```

Nvd response DTO
```
@Data
public class NvdResponseDto {

    private List<Vulnerability> vulnerabilities;

    @Data
    public static class Vulnerability {
        private CveData cve;
    }

    @Data
    public static class CveData {
        private String id;
        private String published;
        private String lastModified;
        private List<Description> descriptions;
        private Metrics metrics;
    }

    @Data
    public static class Description {
        private String value;
    }

    @Data
    public static class Metrics {
        private List<CvssMetricV31> cvssMetricV31;
    }

    @Data
    public static class CvssMetricV31 {
        private CvssData cvssData;
    }

    @Data
    public static class CvssData {
        private Double baseScore;
    }
}

```

json to xml

```
package com.example.demo.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.xml.XmlMapper;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Service;

import java.io.File;

@Service
public class JsonToXmlService {

    public void convertJsonFileToXml() throws Exception {

        // Read JSON file from resources
        ClassPathResource resource = new ClassPathResource("data.json");

        ObjectMapper jsonMapper = new ObjectMapper();
        Object obj = jsonMapper.readValue(resource.getInputStream(), Object.class);

        // Convert to XML
        XmlMapper xmlMapper = new XmlMapper();
        xmlMapper.writerWithDefaultPrettyPrinter()
                 .writeValue(new File("output.xml"), obj);

        System.out.println("Conversion Completed!");
    }
}
```
