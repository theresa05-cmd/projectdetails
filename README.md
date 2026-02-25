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


