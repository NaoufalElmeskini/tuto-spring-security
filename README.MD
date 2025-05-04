## intro :

Spring est (grandement) basé sur des singleton. des instances de class qui sont partagées dans toutes l'application.
dans le cas d'une API REST, cela veut dire que deux utilisateurs, partagent la même instance d'un controller.
cela rend évident les traitements génériques, mais comment récupérer des informations d'utilisateurs spécifiques pour un traitement plus poussé ? comment garantir la confidentialité de ces données ?
ex : Spring Security et SecurityContextHolder

### niveau 1 : traitement générique

scenario 1 :
en tant qu'utilisateur quelconque
lorsque j'envois une requete HTTP : GET /bienvenue
je veux recevoir "bienvenue !"

## implementation

grâce à la magie de Spring Boot, seule dependance necessaire :
``` xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-web</artifactId>  
</dependency>
```

ensuite le controller et methode de mapping à la requête HTTP :
``` java
@RestController  
public class AccueilController {  
    @GetMapping("/bienvenue")  
    public String accueillir() {  
       return "bienvenue !";  
    }  
}
``` 


résultat du test : (check)

![Paste![Pasted image 20250504143219.png](annexe/Pasted%20image%2020250504143219.png)d image 20250504143219.png](Pasted%20image%2020250504143219.png)



#### explication :
derrière le rideau, voici ce qui se passe :
1. l'annotation @RestController permet à Spring Boot de marquer la classe AccueilController comme bean. ainsi : lors du démarrage de l'application, Spring crée **une seule** instance de la class AccueilController. concrètement : cette instance est crée  et enregistré en tant que (bean) dans (l'objet representant tout le contexte Spring : ApplicationContexte) (appelons cette instance "aControllerSingleton").

```
pourquoi 1 seule instance ? cela vient de l'annotation @RestController qui marque la classe. derrière, @RestController utilise @Component, un des 6 stéréotypes Spring. cette annotation indique au contexte Spring d'utiliser un singleton par defaut.
```
2. toujours lors du démarrage : Spring crée un mapping entre la requête HTTP "GET /bienvenue" la methode AccueilController::accueillir (+ref)
3. lors de notre test, la requête "GET /bienvenue" est reçue par l'application. grace au mapping de l'étape precedante, la méthode de l'instance aControllerSingleton

```
pour que la requête arrive jusqu'à la methode, Spring utilise le design pattern FrontController, notemment via la class DispatcherServlet. elle en fait de la route !
```
![Pasted image 20250504145905.png](annexe/Pasted%20image%2020250504145905.png)


## niveau 2 : traitement spécifique par utilisateur

scénario 2A :
en tant qu'utilisateur Aboubakar
lorsque j'envois une requete HTTP : GET /bienvenue
je veux recevoir "bienvenue Aboubakar !"

scénario 2B :
en tant qu'utilisateur Béatrice
lorsque j'envois une requete HTTP : GET /bienvenue
je veux recevoir "bienvenue Béatrice !"

#### implementation
une notion d'utilisateur apparait !
spoiler : Spring Security à la rescousse !
dans les grandes lignes, voici le plan :
- .. adapter le socle
- a. déclarer des utilisateurs pouvant se connecter à notre application
- b. protéger nos endpoints sensible contre les utilisateurs inconnus
- c. authentifier des utilisateur lors d'appel
- d. recuperer les information de l'utilisateur

##### 0. adapter le socle
pour ajouter spring security, il suffit d'ajouter la dependance suivante dans le pom.xml.
``` xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-security</artifactId>  
</dependency>
```
attention : dans mon projet, j'utilise la version 3.3.x de Spring Boot. il se peut que d'autres étapes soient necessaire si vous utilisez des versions antérieures. (ajouter une configuration @EnableWebSecurity...)

##### 1. déclarer des utilisateurs
attention : jamais en production ! dans cette application de demo, il suffit de déclarer en dur 2 utilisateurs de test : Aboubakar et Béatrice
```
@Configuration  
public class SecurityConfig {  
    @Bean  
    public UserDetailsService users() {  
       return new InMemoryUserDetailsManager(  
             User.builder()  
                   .username("beatrice")  
                   .password("{noop}password")  
                   .build(),  
             User.builder()  
                   .username("aboubakar")  
                   .password("{noop}password")  
                   .build()  
       );  
    }
}
```


##### 2. protéger les endpoints
en réalité, nul besoin de faire un changement pour protéger nos endpoints. en effet, le simple fait d'ajouter la dependance spring-security au projet, ferme les endpoints de l'application aux utilisateurs non authentifiée ! voici ce que dit la [documentation](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/boot-features-security.html) :
```
If [Spring Security](https://projects.spring.io/spring-security/) is on the classpath, then web applications are secured by default.
```
![Pasted image 20250504152901.png](annexe/Pasted%20image%2020250504152901.png)

##### 3. authentifier des utilisateur lors d'appel

HTTP headers ! si vous utilisez un outil de requête HTTP comme Insomnia ou Postman, il suffit de renseigner l'onglet "Auth". cela crée automatiquement un header "Authorization" avec la valeur  
"Basic XXX" :
![Pasted image 20250504153753.png](annexe/Pasted%20image%2020250504153753.png)
```
- "Basic " : indique la methode d'authentification utilisée ([Basic](https://en.wikipedia.org/wiki/Basic_access_authentication), [Bearer](https://en.wikipedia.org/wiki/OAuth)...)
- l'effrayante chaine "YWJvdWJha2FyOnBhc3N3b3Jk" n'est en fait que notre paire user/password, encodé en base64.
```

##### 4. récuperer les informations de l'utilisateur

``` java
@RestController  
public class UserDataController {  
  
    @GetMapping("/quisuisje")  
    public ResponseEntity<String> whoami() {  
       UserDetails userConnecte = (UserDetails) SecurityContextHolder  
             .getContext()  
             .getAuthentication()  
             .getPrincipal();  
       return ResponseEntity.ok(userConnecte.getUsername());  
    }  
}
```
