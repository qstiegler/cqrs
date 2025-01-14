# Reduced cost of change through CQRS in Symfony

## Installation and configuration

Install package through composer:

```shell
composer require digital-craftsman/cqrs
```

Then add the following `cqrs.yaml` file to your `config/packages` and replace it with your instances of the interfaces:

```yaml
cqrs:

  query_controller:
    default_request_decoder_class: 'DigitalCraftsman\CQRS\RequestDecoder\JsonRequestDecoder'
    default_dto_constructor_class: 'DigitalCraftsman\CQRS\DTOConstructor\SerializerDTOConstructor'
    default_response_constructor_class: 'DigitalCraftsman\CQRS\ResponseConstructor\SerializerJsonResponseConstructor'

  command_controller:
    default_request_decoder_class: 'DigitalCraftsman\CQRS\RequestDecoder\JsonRequestDecoder'
    default_dto_constructor_class: 'DigitalCraftsman\CQRS\DTOConstructor\SerializerDTOConstructor'
    default_response_constructor_class: 'DigitalCraftsman\CQRS\ResponseConstructor\EmptyResponseConstructor'

```

You can find the [full configuration here](./docs/configuration.md). 

The package contains instances for request decoder, DTO constructor and response constructor. With this you can already use it. You only need to create your own DTO validators, DTO data transformers and handler wrappers when you want to use those. 

Where and how to use the instances, is described below.

## Why

It's very easy to build a CRUD and REST API with Symfony. There are components like parameter converter which are all geared towards getting data very quickly into a controller to handle the logic there. Unfortunately even though it's very fast to build endpoints with a REST mindset, it's very difficult to handle business logic in a matter that makes changes easy and secure. In short, we have a **[low cost of introduction at the expense of the cost of change](https://www.youtube.com/watch?v=uQUxJObxTUs)**.

The CQRS construct closes this gap and **drastically reduces the cost of change** without much higher costs of introduction.

### Overarching goals

The construct has to following goals:

1. Make it very fast and easy to understand **what** is happening (from a business logic perspective).
2. Make the code safer through extensive use of value objects.
3. Make refactoring safer through the extensive use of types.
4. Add clear boundaries between business logic and application / infrastructure logic.

### How

The construct consists of two starting points, the `CommandController` and the `QueryController` and the following components:

- **[Request decoder](./docs/request-decoder.md)**  
*Parses the request and transforms it into an array structure.*
- **[DTO data transformer](./docs/dto-data-transformer.md)**  
*Transforms the previously generated array structure if necessary.*
- **[DTO constructor](./docs/dto-constructor.md)**  
*Generates a command or query from the array structure.*
- **[DTO validator](./docs/dto-validator.md)**  
*Validates the created command or query.*
- **[Handler](./docs/handler.md)**  
*Command or query handler which contains the business logic.*
- **[Handler wrapper](./docs/handler-wrapper.md)**  
*Wraps handler to execute logic as a prepare / try / catch logic.*
- **[Response constructor](./docs/response-constructor.md)**  
*Transforms the gathered data of the handler into a response.*

Through the Symfony routing, we define which instances of the components (if relevant) are used for which route. We use PHP files for the routes instead of the default YAML for more type safety and so that renaming of components is easier through the IDE.

A route might look like this:

```php
$routes->add(
    'api_news_create_news_article_command',
    '/api/news/create-news-article-command',
)
    ->controller([CommandController::class, 'handle'])
    ->methods([Request::METHOD_POST])
    ->defaults([
        'routePayload' => Configuration::routePayload(
            dtoClass: CreateNewsArticleCommand::class,
            handlerClass: CreateNewsArticleCommandHandler::class,
            dtoValidatorClasses: [
                UserIdValidator::class,
            ],
        ),
    ]);
```

You only need to define the components that differ from the defaults configured in the `cqrs.yaml` configuration. Read more about [routing here](./docs/routing.md).

### Command example

Commands and queries are strongly typed value objects which already validate whatever they can. Here is an example command that is used to create a news article:

```php
<?php

declare(strict_types=1);

namespace App\Domain\News\WriteSide\CreateNewsArticle;

use App\Helper\HtmlHelper;
use App\ValueObject\UserId;
use Assert\Assertion;
use DigitalCraftsman\CQRS\Command\Command;

final class CreateNewsArticleCommand implements Command
{
    public function __construct(
        public UserId $userId,
        public string $title,
        public string $content,
        public bool $isPublished,
    ) {
        Assertion::betweenLength($title, 1, 255);
        Assertion::minLength($content, 1);
        HtmlHelper::assertValidHtml($content);
    }
}

```

The structural validation is therefore already done through the creation of the command and the command handler only has to handle the business logic validation. A command handler might look like this: 

```php
<?php

declare(strict_types=1);

namespace App\Domain\News\WriteSide\CreateNewsArticle;

use App\DomainService\UserCollection;
use App\Entity\NewsArticle;
use App\Time\Clock\ClockInterface;
use App\ValueObject\NewsArticleId;
use DigitalCraftsman\CQRS\Command\Command;
use DigitalCraftsman\CQRS\Command\CommandHandlerInterface;
use Doctrine\ORM\EntityManagerInterface;

final class CreateNewsArticleCommandHandler implements CommandHandlerInterface
{
    public function __construct(
        private ClockInterface $clock,
        private UserCollection $userCollection,
        private EntityManagerInterface $entityManager,
    ) {
    }

    /** @param CreateProductNewsArticleCommand $command */
    public function handle(Command $command): void
    {
        $commandExecutedAt = $this->clock->now();

        // Validate
        $requestingUser = $this->userCollection->getOne($command->userId);
        $requestingUser->mustNotBeLocked();
        $requestingUser->mustHavePermissionToWriteArticle();

        // Apply
        $this->createNewsArticle($command, $commandExecutedAt);
    }

    private function createNewsArticle(
        CreateProductNewsArticleCommand $command,
        \DateTimeImmutable $commandExecutedAt,
    ): void {
        $newsArticleId = NewsArticleId::generateRandom();
        $newsArticle = new NewsArticle(
            $newsArticleId,
            $command->title,
            $command->content,
            $command->isPublished,
            $commandExecutedAt,
        );

        $this->entityManager->persist($newsArticle);
        $this->entityManager->flush();
    }
}
```
