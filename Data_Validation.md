# Data Validation

The Symfony validator is a powerful tool that can be leveraged to guarantee that the data of any object is "valid". The power behind validation lies in "constraints", which are rules that you can apply to properties or getter methods of your object. And while you'll most commonly use the validation framework indirectly when using forms, remember that it can be used anywhere to validate any object.

## PHP Object Validation

* The `debug:validator` command can be used to see which validation constraints are applied to a given class.
* You have to use the validator service (which implements ValidatorInterface) with the validate() method (i.e. inside a controller) so as it reads the constraints of the class and check if the object satisfies those constraints and returns the potential errors.
**Outside the context of form validation, you have to explicitly call this service**<br>
`$errors = $validator->validate($object);` 
* It's also possible to use the Validation callables :
- createCallable()
This returns a closure that throws ValidationFailedException when the constraints aren't matched.
```php
use Symfony\Component\Validator\Constraints\Regex;
use Symfony\Component\Validator\Validation;

$question = new Question('Please enter the name of the bundle', 'AcmeDemoBundle');
$validation = Validation::createCallable(new Regex([
    'pattern' => '/^[a-zA-Z]+Bundle$/',
    'message' => 'The name of the bundle should be suffixed with \'Bundle\'',
]));
$question->setValidator($validation);
```
- createIsValidCallable()
This returns a closure that returns false when the constraints aren't matched.

```php
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\Validator\Constraints\Length;
use Symfony\Component\Validator\Validation;

// ...
$resolver->setAllowedValues('transport', Validation::createIsValidCallable(
    new Length(['min' => 10 ])
));
```

## Built-in Constraints

[Full list and details](https://symfony.com/doc/current/validation.html#constraints)

## Validation Scopes

* You can target properties, *getters* (any private, protected or public method whose name starts with "get", "is" or "has") or the whole class (some built-in contraints like [Callback](https://symfony.com/doc/current/reference/constraints/Callback.html) apply to the whole class).
* You can target an entire class with a custom validation constraint by overriding the getTargets method in your class :
```php
// src/Validator/ConfirmedPaymentReceipt.php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;

#[\Attribute]
class ConfirmedPaymentReceipt extends Constraint
{
    public string $userDoesNotMatchMessage = 'User\'s e-mail address does not match that of the receipt';

    public function getTargets(): string
    {
        return self::CLASS_CONSTRAINT;
    }
}
```
* In your class inherits from another class, **the constraints defined in the parent properties will be applied to the child properties even if the child properties override those constraints.**
To change this behaviour, we can use validation groups

## Validation Groups

```php
// src/Entity/User.php
namespace App\Entity;

use Symfony\Component\Security\Core\User\UserInterface;
use Symfony\Component\Validator\Constraints as Assert;

class User implements UserInterface
{
    #[Assert\Email(groups: ['registration'])]
    private string $email;

    #[Assert\NotBlank(groups: ['registration'])]
    #[Assert\Length(min: 7, groups: ['registration'])]
    private string $password;

    #[Assert\Length(min: 2)]
    private string $city;
}
```
Here we have three validation groups here :
- `User` : Based on the name of the class, it checks for all the constraints that are not inside any other validation groups. If there was an embedded object in a property (i.e. $address from an Address class) on which there was the Valid constraints, then this group will **NOT** validate this embedded object's properties that have not the `User` group on them.
- `Default` : Same as User. Must not be used in a GroupSequence because it will cause an infinite loop
- `registration` : A custom validation group

* Validation groups could be specified in the 3rd argument of the validate() method (from the Validator service), or with the `validation_groups` option when dealing with forms (with `createFormBuilder` or `configureOptions` methods).

## Group Sequence

* You can sequentially apply validation rules using GroupSequence.
* When using GroupSequence, the `Default` group will now reference the group sequence, instead of all constraints that do not belong to any group.
* You can use GroupSequence with `validation_groups` form option.
```php
public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'validation_groups' => new GroupSequence(['First', 'Second']),
        ]);
    }
```
* We can dynamically determine which group sequence to use with GroupSequenceProvider
by implementing GroupSequenceProviderInterface (and so defining `getGroupSequence` method) and notifying the Validator that our class provides a GroupSequence with the attribute :<br> `#[Assert\GroupSequenceProvider]`
```php
// src/Entity/User.php
namespace App\Entity;

use Symfony\Component\Validator\Constraints as Assert;

#[Assert\GroupSequenceProvider]
class User implements GroupSequenceProviderInterface
{
    #[Assert\NotBlank]
    private string $name;

    #[Assert\CardScheme(
        schemes: [Assert\CardScheme::VISA],
        groups: ['Premium'],
    )]
    private string $creditCard;

    // ...

    public function getGroupSequence(): array|GroupSequence
    {
        // when returning a simple array, if there's a violation in any group
        // the rest of the groups are not validated. E.g. if 'User' fails,
        // 'Premium' and 'Api' are not validated:
        return ['User', 'Premium', 'Api'];

        // when returning a nested array, all the groups included in each array
        // are validated. E.g. if 'User' fails, 'Premium' is also validated
        // (and you'll get its violations too) but 'Api' won't be validated:
        return [['User', 'Premium'], 'Api'];
    }
}
```

* You can separate the logic by creating a class dedicated to it (which will implements GroupUserProviderInterface and define getGroupSequence) and referencing this class inthe targeted entity with the provider option :
```php
// src/Entity/User.php
namespace App\Entity;

// ...
use App\Validator\UserGroupProvider;

#[Assert\GroupSequenceProvider(provider: UserGroupProvider::class)]
class User
{
    // ...
}
```

* Sometimes, you will want to sequentially apply constraints on a specific property. It will be more straight-forward to use the [Sequentially](https://symfony.com/doc/current/reference/constraints/Sequentially.html) constraint.

## The violation builder

[Here](https://github.com/symfony/symfony/blob/7.1/src/Symfony/Component/Validator/Violation/ConstraintViolationBuilder.php#L27)