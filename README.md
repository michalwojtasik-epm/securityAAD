# Set-up Spring Security z Azure Active Directory

###### Garść przydatnych linków:
1. [Strona azure - ogłoszenie integracji z Spring Security 5.0](https://azure.microsoft.com/pl-pl/blog/use-azure-active-directory-with-spring-security-5-0-for-oauth-2-0/)
2. [Github Azure - przykładowa aplikacja z autentyfikacją pod stronie frontu](https://github.com/Microsoft/azure-spring-boot/tree/master/azure-spring-boot-samples/azure-active-directory-spring-boot-sample)
3. [Ładny obrazek z zasadą działania autoryzacji AAD](https://github.com/microsoft/azure-spring-boot/tree/master/azure-spring-boot-starters/azure-active-directory-spring-boot-starter)

## Ogólne informacje

### [ADAL](https://docs.microsoft.com/pl-pl/azure/active-directory/develop/active-directory-authentication-libraries) oraz [MSAL](https://docs.microsoft.com/pl-pl/azure/active-directory/develop/msal-overview)
Interesującą nas opcją jest autoryzacja po stronie Front-endu, a następnie wysyłanie autoryzowanych zapytań do backendu. Istnieją dwie Microsoftowe biblioteki do autoryzacji przy pomocy JS, [ADAL](https://docs.microsoft.com/pl-pl/azure/active-directory/develop/active-directory-authentication-libraries) oraz [MSAL](https://docs.microsoft.com/pl-pl/azure/active-directory/develop/msal-overview) (przy czym na githubie na stronie jakiegoś issue ich developer napisał iż 'zalecają' używanie MSAL, gdyż od ADAL odchodzą - niestety na żadnej innej stronie tego nie widziałem, więc nie wiem czy potwierdzone info).

### [react-aad-msal](https://www.npmjs.com/package/react-aad-msal)
Dodatkowo istnieje open source'owa bibliteka do reacta 'opakowująca' bibliotekę MSAL - [react-aad-msal](https://www.npmjs.com/package/react-aad-msal). Wygląda bardzo elegancko i prosto - dostępna również na ich GitHubie [przykładowa aplikacja](https://github.com/syncweek-react-aad/react-aad/tree/master/samples/react-javascript)

### Rejestracja na Azure, oraz zapytanie do API
Gdy mamy poprawnie [zarejestrowaną aplikację](https://docs.microsoft.com/en-us/azure/java/spring-framework/configure-spring-boot-starter-java-app-with-azure-active-directory) na Azure(! Chyba jeszcze nie mamy - parametr `oauth2AllowImplicitFlow` w manifescie powinien być ustawiony na wartość `true`, przynajmniej według strony [tutaj](https://docs.microsoft.com/en-us/azure/java/spring-framework/configure-spring-boot-starter-java-app-with-azure-active-directory), a niestety na starym nmarkecie jest na false - więc CHYBA trzeba zarejestrować nową), po autoryzacji na froncie będziemy mieli dostęp do AccessID - id dzięki któremu możemy tworzyć autoryzowane zapytania do API - wystarczy do zapytania dodać nagłówek: 
>"Authorization" : "Bearer " + AccessID

### Spring Security razem z Azure AD
Konfiguracja podstawowa jest bardzo prosta, opisana w readme [TUTAJ](https://github.com/microsoft/azure-spring-boot/tree/master/azure-spring-boot-starters/azure-active-directory-spring-boot-starter), razem z przykładową aplikacją. Obejmuje:

1. Konfigurację Azure AD w `application.properties` :
```
azure.activedirectory.client-id=Application-ID-in-AAD-App-registrations
azure.activedirectory.client-secret=Key-in-AAD-API-ACCESS
azure.activedirectory.active-directory-groups=Aad-groups e.g. group1,group2,group3
```

2. Wstrzyknięcie filtru `aadAuthFilter`
```
@Autowired
private AADAuthenticationFilter aadAuthFilter;
```

3. Następnie można skorzystać z adnotacji `@PreAuthorize("hasRole('GROUP_NAME')")` przed odpowiednią metodą kontrolera, lub w klasie konfiguracyjnej sprawić że wszystkie zapytania muszą być autoryzowane:
```
@Override
    protected void configure(HttpSecurity http) throws Exception {
            .authorizeRequests()
              .anyRequest()
              .authenticated()
    }
```

## Aktualne zmiany

Aktualnie teoretycznie na branchu są zmiany w serwerze, które powinny dawać autoryzację tylko zapytaniom z UI które mają poprawny AccessID. Praktycznie serwer poprawnie odrzuca nieautoryzowane zapytania, niestety te poprawne powodują wyrzucenie błędu:
```
com.nimbusds.jwt.proc.BadJWTException: Invalid token issuer
	at com.microsoft.azure.spring.autoconfigure.aad.UserPrincipal$1.verify(UserPrincipal.java:123) ~[azure-spring-boot-2.0.4.jar:na]
	at com.nimbusds.jwt.proc.DefaultJWTProcessor.verifyAndReturnClaims(DefaultJWTProcessor.java:261) ~[nimbus-jose-jwt-6.0.2.jar:6.0.2]
	at com.nimbusds.jwt.proc.DefaultJWTProcessor.process(DefaultJWTProcessor.java:342) ~[nimbus-jose-jwt-6.0.2.jar:6.0.2]

```
Co wydaje mi się, iż jest spowodowane własnie tym iż aktualnie korzystamy z ClientID oraz SecretKey starego nMarketu zarejestrowanego w Azure AD, który parametr `oauth2AllowImplicitFlow` w manifescie ma ustawiony na wartość `false` zamiast na `true`. Jednakże istnieje zapewne duża szansa iż jestem w błędzie i problem leży gdzie indziej.
