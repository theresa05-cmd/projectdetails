# projectdetails

1. whenever a project given with json data then to parse and load on to database
first create an entity 
here in entity it should be based on what we are giving in the sql table 
sample entity as follows

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

