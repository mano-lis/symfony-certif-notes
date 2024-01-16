# Security

## Authentication

Symfony SecurityBundle comes with several authenticators :

### Form login

* Provided by [FormLoginAuthenticator](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/FormLoginAuthenticator.php) which handles automatically the form submission
You only need a piece of config and a controller :
```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        main:
            # ...
            form_login:
                # "app_login" is the name of the route created previously
                login_path: app_login
                check_path: app_login
```
```php
+ use Symfony\Component\Security\Http\Authentication\AuthenticationUtils;

  class LoginController extends AbstractController
  {
     #[Route('/login', name: 'app_login')]
     public function index(AuthenticationUtils $authenticationUtils): Response
     {
         // get the login error if there is one
         $error = $authenticationUtils->getLastAuthenticationError();

         // last username entered by the user
         $lastUsername = $authenticationUtils->getLastUsername();

         return $this->render('login/index.html.twig', [
             'last_username' => $lastUsername,
             'error'         => $error,
         ]);
     }
 }
```
* the username field in the form template must have the name `_username` and password field `_password`
* You can add a CSRF field (mandatory name `_csrf_token` and add `form_login.enable_csrf: true` in security.yaml)

### JSON Login

### HTTP Basic

### Login Link

### Access Token

* You just have to define a handler (which implements [AccessTokenHandlerInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/AccessToken/AccessTokenHandlerInterface.php))
```yaml
# config/packages/security.yaml
security:
    firewalls:
        main:
            access_token:
                token_handler: App\Security\AccessTokenHandler
```
```php
// src/Security/AccessTokenHandler.php
namespace App\Security;

use App\Repository\AccessTokenRepository;
use Symfony\Component\Security\Http\AccessToken\AccessTokenHandlerInterface;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;

class AccessTokenHandler implements AccessTokenHandlerInterface
{
    public function __construct(
        private AccessTokenRepository $repository
    ) {
    }

    public function getUserBadgeFrom(string $accessToken): UserBadge
    {
        // e.g. query the "access token" database to search for this token
        $accessToken = $this->repository->findOneByValue($accessToken);
        if (null === $accessToken || !$accessToken->isValid()) {
            throw new BadCredentialsException('Invalid credentials.');
        }

        // and return a UserBadge object containing the user identifier from the found token
        return new UserBadge($accessToken->getUserId());
    }
}
```

### X.509 Client Certificates

### Remote Users

* You can also create Custom Authenticators ([see below](#authenticators-passports-and-badges))
* You can customize success and failure handlers on authentication by implementing [AuthenticationSuccessHandlerInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authentication/AuthenticationSuccessHandlerInterface.php) or [AuthenticationFailureHandlerInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authentication/AuthenticationFailureHandlerInterface.php)

## Authorization

Authorization has 2 different sides :
1. The user receives a specific role when logging in (e.g. ROLE_ADMIN).
2. You add code so that a resource (e.g. URL, controller) requires a specific "attribute" (e.g. a role like ROLE_ADMIN) in order to be accessed.

Rather than using access_control option, you can deny access from your code.
* Inside a Controller, you can use `$this->denyAccessUnlessGranted()` or the #[IsGranted()] attribute. If access denied it will throw an AccessDEniedException. After that the user will be redirected to the login page or he'll see a 403 if he was already logged in.
* Inside the twig template, you can use `is_granted()` method (e.g. inside an if loop).
* Access can be checked everywhere in our code by injecting Security object.
* There are several built-in attributes :
    - `IS_AUTHENTICATED` isn't a role, but it kind of acts like one, and every user that has logged in will have this. 
    - `IS_AUTHENTICATED_FULLY`: This is similar to IS_AUTHENTICATED_REMEMBERED, but stronger. Users who are logged in only because of a "remember me cookie" will have
    - `IS_AUTHENTICATED_REMEMBERED` but will not have IS_AUTHENTICATED_FULLY.
    - `IS_REMEMBERED`: Only users authenticated using the remember me functionality, (i.e. a remember-me cookie).
    - `IS_IMPERSONATOR`: When the current user is impersonating another user in this session, this attribute will match.
* On every request (unle)
### Roles

* Roles are stored in db and every user have at least `ROLE_USER`
* Each role must begin with ROLE_ prefix
* You can use role inheritance :
```yaml
# config/packages/security.yaml
security:
    # ...

    role_hierarchy:
        ROLE_ADMIN:       ROLE_USER
        ROLE_SUPER_ADMIN: [ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]
```
* So that role hierarchy is taken into account, we use isGranted() and not getRoles()


### Access Control Rules

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        # ...
        main:
            # ...

    access_control:
        # require ROLE_ADMIN for /admin*
        - { path: '^/admin', roles: ROLE_ADMIN }

        # or require ROLE_ADMIN or IS_AUTHENTICATED_FULLY for /admin*
        - { path: '^/admin', roles: [IS_AUTHENTICATED_FULLY, ROLE_ADMIN] }

        # the 'path' value can be any valid regular expression
        # (this one will match URLs like /api/post/7298 and /api/comment/528491)
        - { path: ^/api/(post|comment)/\d+$, roles: ROLE_USER }
```

* Beginning by the top, only one match by request (between the given url and the regular expression)
* `access_control` has two types of options :
  1. **Matching options** (does request match this entry?) :
    `path`: a regular expression (without delimiters)
    `ip` or `ips`: netmasks are also supported (can be a comma-separated string)
    `port`: an integer
    `host`: a regular expression
    `methods`: one or many HTTP methods
    `request_matcher`: a service implementing RequestMatcherInterface
    `attributes`: an array, which can be used to specify one or more request attributes that must match exactly
    `route`: a route name
  2. **Access Enforcement** (Should the user come in?) :
    `roles`: If the user does not have the given role, then access is denied (internally, an AccessDeniedException is thrown).
    `allow_if`: If the expression returns false, then access is denied;
    `requires_channel`: If the incoming request's channel (e.g. http) does not match this value (e.g. https), the user will be redirected (e.g. redirected from http to https, or vice versa).
**By default, if roles and allow_if are both defined, access will be granted if at least one condition is valid**
It's possible to change this behaviour by modifying Access Decision Strategy ([see Voters](#voters-and-voting-strategies))

## Providers

* The user provider reloads user from a storage. `property` is the user identifier.
* Basic config when using the `make:user` command :
```yaml
# config/packages/security.yaml
security:
    # ...

    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email
```
* On each request the user is loaded from the session (unless `firewall` is stateless) to "refresh" the user and see if all infos are up to date. 
* There four built-in providers in Symfony :
  - **Entity User Provider** : Loads users from a database using Doctrine (If you want more control, you can implement UserLoaderInterface in your UserRepository to create a custom query in `loadUserByIdentifier()` method and remove the property option in security.yaml)
  - **LDAP User Provider** :  Loads users from a LDAP server
  - **Memory User Provider** : Loads users from a configuration file
  - **Chain User Provider** : Merges two or more user providers into a new user provider. Since each firewall has exactly one user provider, you can use this to chain multiple providers together.
```yaml
# config/packages/security.yaml
security:
    # ...
    providers:
        backend_users:
            ldap:
                # ...

        legacy_users:
            entity:
                # ...

        users:
            entity:
                # ...

        all_users:
            chain:
                providers: ['legacy_users', 'users', 'backend_users']
```
Order is important because Symfony will look for users starting from the first provider and will keep looking for in the other providers until the user is found
* All user providers follow this pattern for their service ID: `security.user.provider.concrete.<your-provider-name>`

## Firewalls

A "firewall" is your authentication system: the firewall defines which parts of your application are secured and how your users will be able to authenticate (e.g. login form, API token, etc).

* The firewall is the core of securing your application. Every request within the firewall is checked if it needs an authenticated user. The firewall also takes care of authenticating this user (e.g. using a login form). 
* Only one firewall is active on each request.
* All real URLs are handled by the main firewall (no `pattern` key means it matches all URLs)
* A firewall can have many modes of authentication, and visiting a url under a firwewall does not necessarily mean that authentication is required.
* `lazy` anonymous mode prevent a session from being started if no authorization is needed.
* You can get config with :
```php
$request = $this->requestStack->getCurrentRequest();
$firewallConfig = $this->security->getFirewallConfig($request);
```

## Users

* The User class implements [UserInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Core/User/UserInterface.php)
* A User can be logged in programmatically :
```php
// src/Controller/SecurityController.php
namespace App\Controller\SecurityController;

use App\Security\Authenticator\ExampleAuthenticator;
use Symfony\Bundle\SecurityBundle\Security;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\RememberMeBadge;

class SecurityController
{
    public function someAction(Security $security): Response
    {
        // get the user to be authenticated
        $user = ...;

        // log the user in on the current firewall
        $security->login($user);

        // if the firewall has more than one authenticator, you must pass it explicitly
        // by using the name of built-in authenticators...
        $security->login($user, 'form_login');
        // ...or the service id of custom authenticators
        $security->login($user, ExampleAuthenticator::class);

        // you can also log in on a different firewall...
        $security->login($user, 'form_login', 'other_firewall');

        // ...and add badges
        $security->login($user, 'form_login', 'other_firewall', [(new RememberMeBadge())->enable()]);

        // use the redirection logic applied to regular login
        $redirectResponse = $security->login($user);
        return $redirectResponse;

        // or use a custom redirection logic (e.g. redirect users to their account page)
        // return new RedirectResponse('...');
    }
}
``` 
can also be `logout()`
* A LogoutEvent is dispatched when the user is logged out.
* User can be accessed with `$security->getUser()` and with `app.user` in the template

## Password Hashers

* User class must implement [PasswordAuthenticatedUserInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Core/User/PasswordAuthenticatedUserInterface.php)
* Default password hasher is auto-selected (currently **bcrypt**, other supported are sodium and PBKDF2. You can also create a custom password hasher).
```yaml
# config/packages/security.yaml
security:
    # ...
    password_hashers:
        # Use native password hasher, which auto-selects and migrates the best
        # possible hashing algorithm (which currently is "bcrypt")
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
```
* If your user class is a Doctrine entity and you hash user passwords, the Doctrine repository class related to the user class must implement the PasswordUpgraderInterface.
* When a better new hash algorithm is available, users' password could be rehashed thanks to the `migrate_form` option :
```yaml
# config/packages/security.yaml
security:
    # ...

    password_hashers:
        # a hasher used in the past for some users
        legacy:
            algorithm: sha256
            encode_as_base64: false
            iterations: 1

        App\Entity\User:
            # the new hasher, along with its options
            algorithm: sodium
            migrate_from:
                - bcrypt # uses the "bcrypt" hasher with the default options
                - legacy # uses the "legacy" hasher configured above
```
Using Doctrine, you have to make the UserRepository class implement PasswordUpgraderInterface and define the upgradePassword method :
```php
// src/Repository/UserRepository.php
namespace App\Repository;

// ...
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\PasswordUpgraderInterface;

class UserRepository extends EntityRepository implements PasswordUpgraderInterface
{
    // ...

    public function upgradePassword(PasswordAuthenticatedUserInterface $user, string $newHashedPassword): void
    {
        // set the new hashed password on the User object
        $user->setPassword($newHashedPassword);

        // execute the queries on the database
        $this->getEntityManager()->flush();
    }
}
```

## Authenticators, Passports and Badges

* A custom authenticator should implement [AuthenticatorInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/AuthenticatorInterface.php) or extends [AbstractAuthenticator](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/AbstractAuthenticator.php) (which has a default `createToken()` method suitable for most of use cases)
```php
// src/Security/ApiKeyAuthenticator.php
namespace App\Security;

use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Exception\AuthenticationException;
use Symfony\Component\Security\Core\Exception\CustomUserMessageAuthenticationException;
use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;

class ApiKeyAuthenticator extends AbstractAuthenticator
{
    /**
     * Called on every request to decide if this authenticator should be
     * used for the request. Returning `false` will cause this authenticator
     * to be skipped.
     */
    public function supports(Request $request): ?bool
    {
        return $request->headers->has('X-AUTH-TOKEN');
    }

    public function authenticate(Request $request): Passport
    {
        $apiToken = $request->headers->get('X-AUTH-TOKEN');
        if (null === $apiToken) {
            // The token header was empty, authentication fails with HTTP Status
            // Code 401 "Unauthorized"
            throw new CustomUserMessageAuthenticationException('No API token provided');
        }

        // implement your own logic to get the user identifier from `$apiToken`
        // e.g. by looking up a user in the database using its API key
        $userIdentifier = /** ... */;

        return new SelfValidatingPassport(new UserBadge($userIdentifier));
    }

    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
    {
        // on success, let the request continue
        return null;
    }

    public function onAuthenticationFailure(Request $request, AuthenticationException $exception): ?Response
    {
        $data = [
            // you may want to customize or obfuscate the message first
            'message' => strtr($exception->getMessageKey(), $exception->getMessageData())

            // or to translate this message
            // $this->translator->trans($exception->getMessageKey(), $exception->getMessageData())
        ];

        return new JsonResponse($data, Response::HTTP_UNAUTHORIZED);
    }
}
```
Enable the authenticator :
```yaml
# config/packages/security.yaml
security:

    # ...
    firewalls:
        main:
            custom_authenticators:
                - App\Security\ApiKeyAuthenticator
```
* The authenticate() method extracts credentials and transform these in a [Passport](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/Passport/Passport.php) object.
* On failure you can return a response (e.g. to return a 401) or null (e.g. running the login controller again with the errors). You never use the $exception->getMessage() which might contain sensitive information.

### Passport

* [Passport](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/Passport/Passport.php) contains the user who will be authenticated and some other information (like if "remember me" has to be enabled)
* Use the [UserBadge](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/Passport/Badge/UserBadge.php) to attach the user to the passport. The UserBadge requires a user identifier (e.g. the username or email), which is used to load the user using the user provider: 
```php
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Credentials\PasswordCredentials;

// ...
$passport = new Passport(new UserBadge($email), new PasswordCredentials($plaintextPassword));
```
* [PasswordCredentials](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/Passport/Credentials/PasswordCredentials.php) and [CustomCredentials](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Http/Authenticator/Passport/Credentials/CustomCredentials.php) are supported by default.
* when you don't need any credentials to be checked (e.g. using API tokens), use the SelfValidatingPassport. This class only requires a UserBadge object and optionally Passport Badges.
* You can add an array of other badges as 3rd argument to improve security. (Those ones are supported : RememberMeBadge, PasswordUpgradeBadge, CsrfTokenBadge, PreAuthenticatedUserBadge. PasswordUpgradeBadge is automatically added if Passport has PasswordCredentials.)
* You can set attribute on a passport inside `authenticate()` method.

## Voters and Voting Strategies

Voters are a place to centralize permission logic. When it's not reused or basic, you can manage it from your controller like : 
```php
// src/Controller/PostController.php
// ...

// inside your controller action
if ($post->getOwner() !== $this->getUser()) {
    throw $this->createAccessDeniedException();
}
```
* All voters are called each time you use the isGranted() method on Symfony's authorization checker or call denyAccessUnlessGranted() in a controller (which uses the authorization checker), or by access controls.
* There are 4 voting strategies :
    - `affirmative`: **Default**. This grants access as soon as there is one voter granting access;
    - `consensus`: This grants access if there are more voters granting access than denying. In case of a tie the decision is based on the allow_if_equal_granted_denied config option (defaulting to true);
    - `unanimous`: This only grants access if there is no voter denying access.
    - `priority`: This grants or denies access by the first voter that does not abstain, based on their service priority;
* Here is how to change the voting strategy :
```yaml
# config/packages/security.yaml
security:
    access_decision_manager:
        strategy: unanimous
        allow_if_all_abstain: false
```
* Custom voters must implement [VoterInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Core/Authorization/Voter/VoterInterface.php) or extends [Voter](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Core/Authorization/Voter/Voter.php)
* The #[IsGranted()] attribute or denyAccessUnlessGranted() method (and also the isGranted() method) calls out to the "voter" system.
* Here is an example with a post that can be edit only by his owner and view only if it i public :
```php
// src/Controller/PostController.php

// ...
use Symfony\Component\Security\Http\Attribute\IsGranted;

class PostController extends AbstractController
{
    #[Route('/posts/{id}', name: 'post_show')]
    // check for "view" access: calls all voters
    #[IsGranted('view', 'post')]
    public function show(Post $post): Response
    {
        // ...
    }

    #[Route('/posts/{id}/edit', name: 'post_edit')]
    // check for "edit" access: calls all voters
    #[IsGranted('edit', 'post')]
    public function edit(Post $post): Response
    {
        // ...
    }
}
```
and the voter :
```php
// src/Security/PostVoter.php
namespace App\Security;

use App\Entity\Post;
use App\Entity\User;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class PostVoter extends Voter
{
    // these strings are just invented: you can use anything
    const VIEW = 'view';
    const EDIT = 'edit';

    protected function supports(string $attribute, mixed $subject): bool
    {
        // if the attribute isn't one we support, return false
        if (!in_array($attribute, [self::VIEW, self::EDIT])) {
            return false;
        }

        // only vote on `Post` objects
        if (!$subject instanceof Post) {
            return false;
        }

        return true;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();

        if (!$user instanceof User) {
            // the user must be logged in; if not, deny access
            return false;
        }

        // you know $subject is a Post object, thanks to `supports()`
        /** @var Post $post */
        $post = $subject;

        return match($attribute) {
            self::VIEW => $this->canView($post, $user),
            self::EDIT => $this->canEdit($post, $user),
            default => throw new \LogicException('This code should not be reached!')
        };
    }

    private function canView(Post $post, User $user): bool
    {
        // if they can edit, they can view
        if ($this->canEdit($post, $user)) {
            return true;
        }

        // the Post object could have, for example, a method `isPrivate()`
        return !$post->isPrivate();
    }

    private function canEdit(Post $post, User $user): bool
    {
        // this assumes that the Post object has a `getOwner()` method
        return $user === $post->getOwner();
    }
}
```
* `supports()` check if our voter will vote on this subject/attribute, and if it returns true, then calls `voteOnAttribute()`
