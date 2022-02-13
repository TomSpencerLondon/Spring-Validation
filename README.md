# Related Blog Posts

* [All You Need To Know About Bean Validation With Spring Boot](https://reflectoring.io/bean-validation-with-spring-boot/)

#### Bean Validation Basics
Bean Validation works by defining constraints to the fields of a class by annotating them with certain annotations.

#### Common Validation Annotations
The most common validation annotations include:

- @NotNull: to say that a field must not be null.
- @NotEmpty: to say that a list field must not empty.
- @NotBlank: to say that a string field must not be the empty string (i.e. it must have at least one character).
- @Min and @Max: to say that a numerical field is only valid when it’s value is above or below a certain value.
- @Pattern: to say that a string field is only valid when it matches a certain regular expression.
- @Email: to say that a string field must be a valid email address.

#### Diagram representing this application
![Screenshot 2022-02-13 at 11 56 29](https://user-images.githubusercontent.com/27693622/153751933-38b550aa-8301-44cc-a5aa-856a9dc4b46d.png)

#### Validator
To validate if an object is valid, we pass it into a Validator which checks if the constraints are satisfied:

```java
Set<ConstraintViolation<Input>> violations = validator.validate(customer);
if (!violations.isEmpty()) {
  throw new ConstraintViolationException(violations);
}
```

We can put the @Valid annotation on method parameters and fields to tell Spring that we want a method parameter or field to be validated.

#### Validating Input to a Spring MVC Controller
Let’s say we have implemented a Spring REST controller and want to validate the input that' passed in by a client. 
There are three things we can validate for any incoming HTTP request:

- the request body,
- variables within the path (e.g. id in /foos/{id}) and,
- query parameters.

#### Validating a Request Body
In POST and PUT requests, it’s common to pass a JSON payload within the request body. 
Spring automatically maps the incoming JSON to a Java object. 
Now, we want to check if the incoming Java object meets our requirements.

This is the payload class:

```java

class Input {

  @Min(1)
  @Max(10)
  private int numberBetweenOneAndTen;

  @Pattern(regexp = "^[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}$")
  private String ipAddress;
  
  // ...
}
```

Here we have an int field that must have a value between 1 and 10 inclusively.
We also have a String field that must contain an IP address as defined by the regex in the @Pattern annotation.

To validate the request body of an incoming HTTP request, we annotate the request body with the @Valid annotation in a REST controller:
```java
with the @Valid annotation in a REST controller:

@RestController
class ValidateRequestBodyController {

  @PostMapping("/validateBody")
  ResponseEntity<String> validateBody(@Valid @RequestBody Input input) {
    return ResponseEntity.ok("valid");
  }

}
```
We add the @Valid annotation to the Input parameter, which is also annotated with @RequestBody to mark that it should be 
read from the request body. This tells Spring to pass the object to a Validator before doing anything else.
We then verify the correct behaviour with an integration test:
```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(controllers = ValidateRequestBodyController.class)
class ValidateRequestBodyControllerTest {

  @Autowired
  private MockMvc mvc;

  @Autowired
  private ObjectMapper objectMapper;

  @Test
  void whenInputIsInvalid_thenReturnsStatus400() throws Exception {
    Input input = invalidInput();
    String body = objectMapper.writeValueAsString(input);

    mvc.perform(post("/validateBody")
            .contentType("application/json")
            .content(body))
            .andExpect(status().isBadRequest());
  }
}
```
#### Validating Path Variables and Request Parameters
Validating path variables and request parameters works differently.
Instead of annotating a class field like above, we’re adding a constraint annotation 
(in this case @Min) directly to the method parameter in the Spring controller:

```java
@RestController
@Validated
class ValidateParametersController {

  @GetMapping("/validatePathVariable/{id}")
  ResponseEntity<String> validatePathVariable(
      @PathVariable("id") @Min(5) int id) {
    return ResponseEntity.ok("valid");
  }
  
  @GetMapping("/validateRequestParameter")
  ResponseEntity<String> validateRequestParameter(
      @RequestParam("param") @Min(5) int param) { 
    return ResponseEntity.ok("valid");
  }
}
```
In contrast to request body validation a failed validation will trigger a ConstraintViolationException instead of a MethodArgumentNotValidException. Spring does not register a default exception handler for this exception, 
so it will by default cause a response with HTTP status 500 (Internal Server Error).
To return a 400 status instead we add a custom exception handler to the controller:
```java
@RestController
@Validated
class ValidateParametersController {

  // request mapping method omitted
  
  @ExceptionHandler(ConstraintViolationException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  ResponseEntity<String> handleConstraintViolationException(ConstraintViolationException e) {
    return new ResponseEntity<>("not valid due to validation error: " + e.getMessage(), HttpStatus.BAD_REQUEST);
  }

}
```
We then verify the validation behaviour with an integration test:
```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(controllers = ValidateParametersController.class)
class ValidateParametersControllerTest {

  @Autowired
  private MockMvc mvc;

  @Test
  void whenPathVariableIsInvalid_thenReturnsStatus400() throws Exception {
    mvc.perform(get("/validatePathVariable/3"))
            .andExpect(status().isBadRequest());
  }

  @Test
  void whenRequestParameterIsInvalid_thenReturnsStatus400() throws Exception {
    mvc.perform(get("/validateRequestParameter")
            .param("param", "3"))
            .andExpect(status().isBadRequest());
  }

}
```
#### Validating Input to a Spring Service Method
We can also validate the input to any Spring components. In order to to this, we use a combination of the @Validated and @Valid annotations:
```java
@Service
@Validated
class ValidatingService{

    void validateInput(@Valid Input input){
      // do something
    }

}
```
Again, the @Validated annotation is only evaluated on class level, so don’t put it on a method in this use case.
Here we use a test on the Service:
```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class ValidatingServiceTest {

  @Autowired
  private ValidatingService service;

  @Test
  void whenInputIsInvalid_thenThrowsException(){
    Input input = invalidInput();

    assertThrows(ConstraintViolationException.class, () -> {
      service.validateInput(input);
    });
  }

}
```
#### Handling Validation Errors
When a validation fails, we want to return a meaningful error message to the client.
First, we need to define that data structure. We’ll call it ValidationErrorResponse and it contains a list of Violation objects:
```java
public class ValidationErrorResponse {

  private List<Violation> violations = new ArrayList<>();

  // ...
}

public class Violation {

  private final String fieldName;

  private final String message;

  // ...
}
```
Then, we create a global ControllerAdvice that handles all ConstraintViolationExceptions that bubble up to the controller level.
In order to catch validation errors for request bodies as well, we will also handle MethodArgumentNotValidExceptions:
```java
class ErrorHandlingControllerAdvice {

  @ExceptionHandler(ConstraintViolationException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  @ResponseBody
  ValidationErrorResponse onConstraintValidationException(
      ConstraintViolationException e) {
    ValidationErrorResponse error = new ValidationErrorResponse();
    for (ConstraintViolation violation : e.getConstraintViolations()) {
      error.getViolations().add(
        new Violation(violation.getPropertyPath().toString(), violation.getMessage()));
    }
    return error;
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  @ResponseBody
  ValidationErrorResponse onMethodArgumentNotValidException(
      MethodArgumentNotValidException e) {
    ValidationErrorResponse error = new ValidationErrorResponse();
    for (FieldError fieldError : e.getBindingResult().getFieldErrors()) {
      error.getViolations().add(
        new Violation(fieldError.getField(), fieldError.getDefaultMessage()));
    }
    return error;
  }

}
```