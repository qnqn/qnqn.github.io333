In the [last article](http://kielczewski.eu/2014/04/developing-restful-web-service-with-spring-boot/) I showed how the RESTful web service could be implemented using [Spring Boot](http://projects.spring.io/spring-boot/). In the following I'll make a more classical MVC application out of it, meaning real and pure, CSS-less, HTML form eye-candy.

Since there is not much difference between this and previous RESTful app, I'll focus only on what needs to be changed. And as usual, there is a [source code](https://github.com/bkielczewski/example-spring-boot-mvc) for you to play around on GitHub, and if you want to follow the changes step-by-step, here's the [source code](https://github.com/bkielczewski/example-spring-boot-rest) for the RESTful web service from the previous article. 

## The goal

Just to remind, that since we have a working UserService already implemented with the following interface:

```java
public interface UserService {
    User save(User user);
    List<User> getList();
}
```

And the `User` class defined as this:

```java
// removed JPA annotations for clarity
public class User {

    @NotNull
    @Size(max = 64)
    private String id;
    @NotNull
    @Size(max = 64)
    private String password;

    // getters
}
```

I'm going to show you how to add controllers and views to list and create `User` objects using form. Also, I'm going to show custom form validation and code everything with internationalisation in mind.

## Maven

```xml
<groupId>eu.kielczewski.example.spring</groupId>
<artifactId>example-spring-boot-mvc</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>war</packaging>
```

The new thing is `<packaging>war</packaging>`. The reason for this is that we're going to add some views, and perhaps static resources later, and we want it to be packed once the application gets distributed. No worries, the WAR will still be executable thanks to `spring-boot-maven-plugin`.

Dependency-wise, since we want to use JSP for the views, those two must be present:

```xml
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-jasper</artifactId>
    <scope>provided</scope>
</dependency>

<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>jstl</artifactId>
</dependency>
```

Lack of `<version>` is because it's already defined by `spring-boot-starter-parent` as before. In case you won't be using it, the version tag needs to be added.

## Configuration

Those two lines must be present in `src/main/resources/application.properties`. In the previous example we didn't need this file at all, being happy with the default configuration Spring Boot provided. Now, however, it is the time to show Spring Boot how to find the views:

```
# MVC
spring.view.prefix=/WEB-INF/jsp/
spring.view.suffix=.jsp
```

So they will be stored in `src/main/webapp/WEB-INF/jsp`, and the files will end with `.jsp`. The reason why it's inside `WEB-INF` is because all the contents of `webapp/` are visible by default to the world, except for this directory. So, for example, the file `webapp/test.jsp` could be accessed from the browser on `http://localhost:8080/test.jsp` in a textual form and it may cause a security problem or laughter. For other static resources like JavaScript files or images, that's of course a very desired behaviour.

For people new to Spring Boot - `application.properties` is a file that is registered as a `PropertySource`, and Spring Boot looks for various properties during the auto configuration phase. That allows us to override the defaults. Moreover, depending on `spring.profiles.active` property, Spring Boot will also look for `application-{active profile}.properties, that allows to customize settings for the currently active profile (ie. for development or test environment). As a side note, this files can be written in YAML too if that's a preference.

## User list controller

This one is simple, we'll grab the list of users from the `UserService`. The test looks like this:

```java
@RunWith(MockitoJUnitRunner.class)
public class UserListControllerTest {

    @Mock
    private UserService userService;

    private UserListController userController;

    @Before
    public void setUp() throws Exception {
     userController = new UserListController(userService);
    }

    @Test
    public void shouldGetUserListPage() {
        List<User> userList = stubServiceToReturnExistingUsers(5);
        ModelAndView view = userController.getListUsersView();
        // verify UserService was called once
        verify(userService, times(1)).getList();
        assertEquals("View name should be right", "user_list", view.getViewName());
        assertEquals("Model should contain attribute with the list of users coming from the service",
             userList, view.getModel().get("users"));
    }

    private List<User> stubServiceToReturnExistingUsers(int howMany) {
        List<User> userList = UserUtil.createUserList(howMany);
        when(userService.getList()).thenReturn(userList);
        return userList;
    }

}
```

The logic here is to pretend that the `UserService` returns the list of users. When it does, the list must be in the "users" attribute of the model. The returned view will be called "user_list".

And the implementation:

```java
@Controller
public class UserListController {

    private final UserService userService;

    @Inject
    public UserListController(final UserService userService) {
        this.userService = userService;
    }

    @RequestMapping("/user_list.html")
    public ModelAndView getListUsersView() {
        ModelMap model = new ModelMap();
        model.addAttribute("users", userService.getList());
        return new ModelAndView("user_list", model);
    }

}
```

It's now annotated as a `@Controller`, because we'll be returning views now, and not directly the response body as in RESTful web service. The view will be hooked to the `/user_list.html` URL.

BTW, in some tutorials setting the model attributes sometimes looks like this:

```java
@RequestMapping("/user_list.html")
public String getListUsersView(Model model) {
    model.addAttribute("users", userService.getList());
    return "user_list";
}
```

This is valid, and allows to return just a String with the view name, except I find it emotionally disturbing somehow to manipulate a method parameter ;)

As I'm drifting in this direction already, this could also be written like this:

```java
@ModelAttribute("users")
List<User> getUserList() {
    return userService.getList();
}

@RequestMapping("/user_list.html")
public String getListUsersView() {
    return "user_list";
}
```

Which probably makes sense, especially if there were more methods with the same model attribute.

## User list view

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<!DOCTYPE html>
<html lang="en">
<body>
    <h1><spring:message code="user.list" /></h1>
    <ul>
    <c:forEach items="${users}" var="user">
        <li>
            <c:out value="${user.getId()}" />
        </li>
    </c:forEach>
    </ul>

    <a href="<spring:url value="/user_create.html" />"><spring:message code="user.create" /></a>
</body>
</html>
```

Nothing fancy, just iteration over `users` property of the model, wrapping it in a HTML list. Worth noting is, that the link to `/user_create.html` is generated by the `<spring:url>` tag, which helps if the application were to be deployed into the container with context path different then `/`.

Another thing here is the internationalisation. Instead of writing the `<h1>` contents in plain text, the `<spring:message>` is used. While rendering the view, the message code `user.list` is sought in `messages-{locale
 code}.properties`, or as a fallback, in `messages.properties`. In my example, the file is located in `src/main/resources/` and the property is defined as:

```
user.list=List of users
```

So at this point the listing of the users is finished and it's time to allow creating them by the form.

## Controller to create a user

We need a controller for the view with the form that will allow us to create new user. So, initially the test case will be like this:

```java
@RunWith(MockitoJUnitRunner.class)
public class UserCreateControllerTest {

    private UserCreateController userController;

    @Before
    public void setUp() throws Exception {
        userController = new UserCreateController();
    }

    @Test
    public void shouldGetCreateUserPage() throws Exception {
        ModelAndView view = userController.getCreateUserView();
        assertEquals("View name should be right", "user_create", view.getViewName());
        assertTrue("View should contain attribute with form object", view.getModel().containsKey("form"));
        assertTrue("The form object should be of proper type", view.getModel().get("form") instanceof UserCreateForm);
    }

}
```

It means that after calling `userController.getCreateUserView()` we expect the returned view to be "user_create". Moreover, the model should have "form" attribute, which contains the instance of `UserCreateForm` object.

One may ask, why don't we use `User` object as a form, as in the RESTful method in the [previous article](http://kielczewski.eu/2014/04/developing-restful-web-service-with-spring-boot/). That's because very often the form is different then an object that finally gets stored, it can also can be validated differently. It will be the case here, as the `UserCreateForm` looks like this:

```java
public class UserCreateForm {

    @NotEmpty
    @Size(max = 64)
    private String id;
    @NotEmpty
    @Size(max = 64)
    private String password1;
    private String password2;

    // getters/setters
}
```

It has two fields for the password, to make sure the person typing it won't make a mistake. The `@Size` validation on both fields remains, but `@NotEmpty` is used instead of `@NotNull`. It's Hibernate Validator's custom annotation, and besides testing for nullity it also allows to discard empty strings.

## View with the form to create a user

```
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
<html>
<body>
    <h1><spring:message code="user.create" /></h1>
    <a href="<spring:url value="/user_list.html" />"><spring:message code="user.list" /></a>
    <form:form method="POST" action="/user_create.html" modelAttribute="form">
        <form:errors path="" element="div" />
        <div>
            <form:label path="id"><spring:message code="user.id" /></form:label>
            <form:input path="id" />
            <form:errors path="id" />
        </div>
        <div>
            <form:label path="password1"><spring:message code="user.password1" /></form:label>
            <form:password path="password1" />
            <form:errors path="password1" />
        </div>
        <div>
            <form:label path="password1"><spring:message code="user.password2" /></form:label>
            <form:password path="password2" />
            <form:errors path="password2" />
        </div>
        <div>
            <input type="submit" />
        </div>
    </form:form>
</body>
</html>
```

Since it's not a JSP tutorial, suffice to say it renders a form that is bound to the "form" model attribute :) Though worth noting is that it has two places to display validation errors. One is on top of the form, when the "global" error messages will be displayed, and the other is next to each input.

## Handling the form submission

Now it's time to handle what happens after the form submission. We need to add the following content to the existing test for `UserCreateController`:

```java
@Mock
private UserService userService;
@Mock
private BindingResult result;

@Test
public void shouldCreateUser_GivenThereAreNoErrors_ThenTheUserShouldBeSavedAndUserListViewDisplayed() {
    when(result.hasErrors()).thenReturn(false);
    UserCreateForm form = new UserCreateForm();
    form.setId("id");
    form.setPassword1("password");
    form.setPassword2("password");
    String view = userController.createUser(form, result);
    assertEquals("There should be proper redirect", "redirect:/user_list.html", view);
    ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
    verify(userService, times(1)).save(captor.capture());
    assertEquals(form.getId(), captor.getValue().getId());
    assertEquals(form.getPassword1(), captor.getValue().getPassword());
}
```

This tests a normal form submission. In this case, the `BindingResult` object should return false from `hasErrors()` method. The form is created and passed to userController.createUser() method, together with the mocked result. On successful submission, instead of the view, there should be redirection back to the user list, so it is checked.

The last part is more tricky. This intercepts the argument passed to the `UserService.save()` method, which should be a `User` object correctly filled with the form data. We assume it will be a responsibility of the controller to create this object and populate it.

```java
@Test
public void shouldCreateUser_GivenThereAreFormErrors_ThenTheFormShouldBeDisplayed() {
    when(result.hasErrors()).thenReturn(true);
    String view = userController.createUser(new UserCreateForm(), result);
    verify(userService, never()).save(any(User.class));
    assertEquals("View name should be right", "user_create", view);
}
```

This tests for what happens when the `BindingResults` is going to have errors. The "user_create" view should be returned, so someone may fix the form, and `UserService.save()` should never be called.

```java
@Test
public void shouldCreateUser_GivenThereAlreadyExistUserWithId_ThenTheFormShouldBeDisplayed() {
    when(result.hasErrors()).thenReturn(false);
    when(userService.save(any(User.class))).thenThrow(UserAlreadyExistsException.class);
    String view = userController.createUser(new UserCreateForm(), result);
    verify(result).reject("user.error.exists");
    assertEquals("View name should be right", "user_create", view);
}
```

This tests for what happens if `UserService.save()` throws `UserAlreadyExistsException`. In that case the "global" error message should be set on the form with code `user.error.exists` and the "user_create" view returned to show it and allow for correction.

Having this, the implementation of the `UserCreateController.createUser()` follows:

```java
@RequestMapping(value = "/user_create.html", method = RequestMethod.POST)
public String createUser(@ModelAttribute("form") @Valid UserCreateForm form, BindingResult result) {
    if (result.hasErrors()) {
        return "user_create";
    }
    try {
        userService.save(new User(form.getId(), form.getPassword2()));
    } catch (UserAlreadyExistsException e) {
        result.reject("user.error.exists");
        return "user_create";
    }
    return "redirect:/user_list.html";
}
```

I feel this will need a step by step explanation:

*   It is bound to `/create_user.html` URL and a POST method
*   Takes `UserCreateForm` as a parameter. It is annotated with `@Valid` that fires up checks against the validation constraints on the form (@NotEmpty, @Size). Also it's annotated with `@ModelAttribute`, which exposes this object as an attribute to the model, sparing the need to return a full ModelAndView object from this method, hence it returns only a view name as a String.
*   If `BindingResult` has errors, the "user_create" view is returned.
*   The `UserService` is fed with new `User` object constructed from the form.
*   If `UserAlreadyExistsException` is thrown from the `UserService.save()`, the "global" error is set on the form. The `user.error.exists` is a message code, it will sought in `messages.properties` to render a real message to the user.
*   If everything goes ok, the redirection to `/user_list.html` is returned.

One final remark is that in this case it's probably ok that the controller maps the form to the `User` object as it's very simple. Problem may arise when both the form and destination object will be more complicated or worse - the mapping might involve some sophisticated logic. That may be just too much responsibility for a humble controller and the logic would need to be extracted to a dedicated class.

## Adding custom form validation

At this point it should be working, except the both password fields are not checked whether they match. It's a good case for writing a custom form validator that can have access to all the form data together with a way to influence the `BindingResult` by reporting errors. Such validator will need to implement `org.springframework.validation.Validator` interface, which looks like this:

```java
public interface Validator {
    boolean supports(Class<?> clazz);
    void validate(Object target, Errors errors);
}
```

It's in the Spring docs, but the first method must check whether the validator applies to specific class (in this case UserCreateForm). The second is where the work is done.

The desired behaviour in our case will be tested by this:

```java
@RunWith(MockitoJUnitRunner.class)
public class UserCreateFormPasswordValidatorTest {

    @Mock
    private Errors errors;

    private UserCreateFormPasswordValidator passwordValidator = new UserCreateFormPasswordValidator();

    @Test
    public void shouldSayItSupports_GivenItReceivesUserCreateFormClass() throws Exception {
        assertTrue(passwordValidator.supports(UserCreateForm.class));
    }

    @Test
    public void shouldSayItSupports_GivenItReceivesDifferentClass_ThenShouldReturnFalse() throws Exception {
        assertFalse(passwordValidator.supports(Object.class));
    }

    @Test
    public void shouldValidate_GivenPasswordsMatch_ThenErrorIsNotSet() throws Exception {
        UserCreateForm form = new UserCreateForm();
        form.setPassword1("password");
        form.setPassword2("password");
        passwordValidator.validate(form, errors);
        verify(errors, never()).rejectValue(anyString(), anyString());
    }

    @Test
    public void shouldValidate_GivenPasswordsDoNotMatch_ThenErrorIsSet() throws Exception {
        UserCreateForm form = new UserCreateForm();
        form.setPassword1("password");
        form.setPassword2("different");
        passwordValidator.validate(form, errors);
        verify(errors).rejectValue("password2", "user.error.password.no_match");
    }

}
```

So, if the `validate()` method detects the passwords are different, it should call `Errors.rejectValue()` method setting field-level error message to code `user.error.password.no_match`.

Knowing this, the implementation will be:

```java
@Component
public class UserCreateFormPasswordValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return UserCreateForm.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        UserCreateForm form = (UserCreateForm) target;
        if (!form.getPassword1().equals(form.getPassword2())) {
            errors.rejectValue("password2", "user.error.password.no_match");
        }
    }
}
```

Should be pretty straightforward now. Except one may ask why it's a `@Component`. Because I'd like to show that it can be. It will be a singleton this way, so no need to create it over and over again. It also allows the validator to do some more fancy things using dependency injection, like calling services, etc. Not needed for this one, but maybe someone would like to add validation for duplicate users this way.

Now the `UserCreateController` needs to be told to use it. We will change the constructor to inject it, and the validator needs to be added to the `WebDataBinder`.

```java
@Inject
public UserCreateController(UserService userService, UserCreateFormPasswordValidator passwordValidator) {
    this.userService = userService;
    this.passwordValidator = passwordValidator;
}

@InitBinder("form")
public void initBinder(WebDataBinder binder) {
    binder.addValidators(passwordValidator);
}
```

That's all, done.

## Using Freemarker instead of JSP

**Important**: since Spring Boot 1.1 the Freemarker support is now built-in, see section below.

This one is optional, but since there are many fans of Freemarker as a template engine (myself included)... ~~Spring Boot guys seem not to be, as in the case of JSP is just adding dependencies to Maven and it works, but here it will need to write custom configuration~~.

Maven dependencies (don't forget to remove JSP ones from this project):

```xml
<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
    <version>2.3.20</version>
</dependency>
```

And we need to add a `@Configuration` class:

```java
@Configuration
public class FreemarkerConfig {

    @Value("${spring.view.prefix}")
    private String viewPrefix;

    @Value("${spring.view.suffix}")
    private String viewSuffix;

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath(viewPrefix);
        configurer.setDefaultEncoding("UTF-8");
        return configurer;
    }

    @Bean
    public ViewResolver freeMarkerViewResolver() {
        FreeMarkerViewResolver viewResolver = new FreeMarkerViewResolver();
        viewResolver.setCache(true);
        viewResolver.setPrefix("");
        viewResolver.setSuffix(viewSuffix);
        viewResolver.setContentType("text/html;charset=UTF-8");
        viewResolver.setExposeSpringMacroHelpers(true);
        viewResolver.setExposeRequestAttributes(false);
        viewResolver.setExposeSessionAttributes(false);
        return viewResolver;
    }

}
```

And in `application.properties`:

```
# MVC
spring.view.prefix=/WEB-INF/ftl/
spring.view.suffix=.ftl
```

Done.

## Freemarker in Spring Boot 1.1

Since 1.1 it's even simpler. Make sure to add this to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```

And the configuration (application.properties):

```
# MVC
spring.view.prefix=
spring.view.suffix=.ftl
spring.freemarker.templateLoaderPath=/WEB-INF/ftl
```

## Closing remarks

Hope that helps somepne, feel free to comment and ask questions, play around with the code, and now I need to watch the latest Game of Thrones so if you'd excuse me... :)