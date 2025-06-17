# Code architecture standards at Landbot

At Landbot, we are committed to building scalable, maintainable, and robust software through a clear and well-defined approach to coding and architecture. Our primary goal is to ensure that the code we write not only solves immediate problems but also contributes to a long-term foundation of high-quality software. To achieve this, we embrace a Domain-Centric Approach, commonly known as Domain-Driven Design (DDD), coupled with principles of Clean Architecture.

You can find all the common abstractions and implementations used in this guide in this repository: [landbot-service-scaffolding](https://git.yexir.com/landbot/service-scaffolding/-/tree/main/src/shared?ref_type=heads)

## Why Domain-Driven Design?

Domain-Driven Design (DDD) allows us to align our code with the business’s unique needs by focusing on core business concepts, rules, and logic. By structuring our code around the domain, we create a shared language and understanding between developers and stakeholders, ultimately producing code that is intuitive, expressive, and adaptable to evolving requirements. This approach enhances both developer productivity and the quality of our software as it grows and changes.

## Why Clean Architecture?

Clean Architecture complements DDD by defining clear boundaries and dependencies between layers of our application. This ensures that our domain layer—the heart of our business logic—remains independent of technical concerns like databases, frameworks, and user interfaces. By decoupling these layers, we reduce the ripple effects of changes, making the codebase more resilient to change and easier to maintain over time.

## Purpose of This Guide

This guide provides standards for writing **clean code and structured architecture** in a DDD-centric approach. It covers essential concepts, practices, and patterns that form the backbone of our software development process, such as:

- **Entities, Value Objects, Aggregates, Domain Services and Repositories:** These fundamental DDD elements model our domain with precision and clarity, ensuring that business rules and invariants are rigorously maintained.
- **Application, Domain and Infrastrucure Layers:** We establish clear boundaries for domain logic, service layers, and infrastructure concerns to avoid code duplication, coupling and ensure consistency.
- **Dependency Management:** Following the dependency inversion principle allows us to structure our application such that high-level policies are independent of low-level implementation details.

## Goals of Our Standards

Through these standards, we aim to achieve:

- **Simplicity and Readability:** Code that is easy to understand and follow.
- **Maintainability:** Architecture that is resilient to changes and reduces technical debt.
- **Reusability and Flexibility:** Modular components that can be adapted or reused across different parts of the application.
- **Alignment with Business Needs:** Code that accurately models and adapts to the business domain.

By following the practices in this guide, each team member contributes to a sustainable, high-quality codebase that meets Landbot’s goals for both current and future development. Through adherence to DDD and Clean Architecture, we strive to deliver software that is as robust and adaptable as it is effective in solving business problems.

## Domain Layer

The **Domain Layer** is the core of our application, representing the heart of our business logic and capturing the essential rules that drive our business. This layer defines the core model of the application, ensuring that our software accurately reflects and enforces the business’s unique requirements. Key Elements of the Domain Layer:

- [Entities](#entities)
- [Value Objects](#value-objects)
- [Aggregates](#aggregates)
- [Repositories](#repositories)
- [Domain Services](#domain-services)
- [Domain Events](#domain-events)

### Entities

Represent core business objects with unique identities. These objects are stateful and may undergo transformations according to business rules, always preserving integrity and consistency within the domain.

As a general guideline for writing effective entities:

- Encapsulate domain business logic within the entity.
- Domain validation should be handled by the entity itself.
- Place the `__post_init__` function at the beginning of the entity definition.
- Always create a `new` method for creating new entities. Place it after `__post_init__` method.
- Position business methods (such as `hire` or `is_free_agent`) after `new` method and before validation methods.
- Place validation methods at the end of the entity file.

:no_entry: **Don't do this:**

```python
@dataclass(frozen=True)
class Player(Entity[UUID]):
    name: str
    id: UUID = field(default_factory=uuid4)
```

The example above represents an anemic model, it functions only as a DTO and lacks both domain logic and the expression of core business rules.

:white_check_mark: **Do this:**

```python
@dataclass(frozen=True)
class Player(Entity[UUID]):
    name: str
    id: UUID = field(default_factory=uuid4)
    team_owner: UUID = field(default=None)

    def __post_init__(self) -> None:
        self._validate_id()
        self._validate_name()

    @classmethod
    def new(cls, name: str, team_owner: UUID, id: UUID = uuid4()) -> "Player":
        return cls(
            name=name,
            id=id,
            team_owner=team_owner,
        )

    def is_free_agent(self) -> bool:
        return self.team_owner is None

    def hire(self, team: UUID) -> "Player":
        if not self.is_free_agent():
            raise InvalidDomainError("Player cannot be hired because it is a free agent")
        if self.team_owner == team:
            raise InvalidDomainError(f"Player is already hired by given team {team}")
        return self.clone(free_agent=False, team_owner=team)

    def _validate_id(self) -> None:
        if not isinstance(self.id, UUID):
            raise InvalidDomainError("Player id must be a UUID")

    def _validate_name(self) -> None:
        if self.name is None:
            raise InvalidDomainError('Player name cannot be None')
        if not isinstance(self.name, str):
            raise InvalidDomainError("Player name must be a string")
        if len(self.name) > 150 or len(self.name) < 1:
            raise InvalidDomainError("Player name must be between 1 and 150 characters")

    def _validate_team_owner(self) -> None:
        if self.team_owner is None:
            return
        if not isinstance(self.team_owner, UUID):
            raise InvalidDomainError("Player team owner must be a UUID")
```

The example above includes domain validations, ensuring that the entity’s state is always valid. It also encapsulates core business rules and invariants, making it clean, expressive, and representative of the domain logic.

### Value objects

A value Object is used to encapsulate attributes and validation logic related to a specific concept (or set of concepts) without a unique identity.

:white_check_mark: **You can do this:**

```python
@dataclass(frozen=True)
class PlayerId:
    value: UUID = field(default_factory=uuid4)

    def __post_init__(self):
        if not isinstance(self.value, UUID):
            raise InvalidDomainError("Player ID must be a valid UUID")

@dataclass(frozen=True)
class Player:
    name: str
    id: PlayerId = field(default=PlayerId(value=uuid4())
    team_owner: UUID = field(default=None)

    # __post_init__

    @classmethod
    def new(cls, name: str, team_owner: UUID, id: UUID = uuid4()) -> "Player":
        return cls(
            name=name,
            id=PlayerId(value=id),
            team_owner=team_owner,
        )

    # domain business_methods

    def _validate_name(self) -> None:
        if self.name is None:
            raise InvalidDomainError('Player name cannot be None')
        if not isinstance(self.name, str):
            raise InvalidDomainError("Player name must be a string")
        if len(self.name) > 150 or len(self.name) < 1:
            raise InvalidDomainError("Player name must be between 1 and 150 characters")

    def _validate_team_owner(self) -> None:
        if self.team_owner is None:
            return
        if not isinstance(self.team_owner, UUID):
            raise InvalidDomainError("Player team owner must be a UUID")
```

:white_check_mark: **And this:**

```python
@dataclass(frozen=True)
class EntityId:
    value: UUID = field(default_factory=uuid4)

    def __post_init__(self):
        if not isinstance(self.value, UUID):
            raise InvalidDomainError("Entity ID must be a valid UUID")

@dataclass(frozen=True)
class Player:
    name: str
    id: EntityId = field(default=EntityId(value=uuid4())
    team_owner: UUID = field(default=None)

    # __post_init__

    @classmethod
    def new(cls, name: str, team_owner: UUID, id: UUID = uuid4()) -> "Player":
        return cls(
            name=name,
            id=EntityId(value=id),
            team_owner=team_owner,
        )

    # domain business_methods

    def _validate_name(self) -> None:
        if self.name is None:
            raise InvalidDomainError('Player name cannot be None')
        if not isinstance(self.name, str):
            raise InvalidDomainError("Player name must be a string")
        if len(self.name) > 150 or len(self.name) < 1:
            raise InvalidDomainError("Player name must be between 1 and 150 characters")

    def _validate_team_owner(self) -> None:
        if self.team_owner is None:
            return
        if not isinstance(self.team_owner, UUID):
            raise InvalidDomainError("Player team owner must be a UUID")
```

:white_check_mark: **But also this:**

```python
@dataclass(frozen=True)
class MainPlayerProperties:
    id: UUID = field(default_factory=uuid4)
    name: str

    def __post_init__(self):
        self._validate_id()
        self._validate_name()

    def _validate_id(self) -> None:
        if not isinstance(self.id, UUID):
            raise InvalidDomainError("Player id must be a UUID")

    def _validate_name(self) -> None:
        if self.name is None:
            raise InvalidDomainError('Player name cannot be None')
        if not isinstance(self.name, str):
            raise InvalidDomainError("Player name must be a string")
        if len(self.name) > 150 or len(self.name) < 1:
            raise InvalidDomainError("Player name must be between 1 and 150 characters")

@dataclass(frozen=True)
class Player:
    properties: MainPlayerProperties
    team_owner: UUID = field(default=None)

   # __post_init__

    @classmethod
    def new(cls, name: str, team_owner: UUID, id: UUID = uuid4()) -> "Player":
        return cls(
            properties=MainPlayerProperties(id=id, name=name)
            team_owner=team_owner,
        )

    # domain business_methods


    def validate_team_owner(self) -> None:
        if self.team_owner is None:
            return
        if not isinstance(self.team_owner, UUID):
            raise InvalidDomainError("Player team owner must be a UUID")
```

:white_check_mark: **Or even this:**

```python
@dataclass(frozen=True)
class EntityId:
    value: UUID = field(default_factory=uuid4)

    def __post_init__(self):
        if not isinstance(self.value, UUID):
            raise InvalidDomainError("Entity ID must be a valid UUID")

@dataclass(frozen=True)
class TeamOwner:
    value: EntityId = field(EntityId(value=uuid4()))

@dataclass(frozen=True)
class MainPlayerProperties:
    id: EntityId = field(default=EntityId=(value=uuid4()))
    name: str

    def __post_init__(self):
        self._validate_id()
        self._validate_name()

    def _validate_name(self) -> None:
        if self.name is None:
            raise InvalidDomainError('Player name cannot be None')
        if not isinstance(self.name, str):
            raise InvalidDomainError("Player name must be a string")
        if len(self.name) > 150 or len(self.name) < 1:
            raise InvalidDomainError("Player name must be between 1 and 150 characters")

@dataclass(frozen=True)
class Player:
    properties: MainPlayerProperties
    team_owner: TeamOwner = field(default=None)

   # __post_init__

    @classmethod
    def new(cls, name: str, team_owner: UUID, id: UUID = uuid4()) -> "Player":
        return cls(
            properties=MainPlayerProperties(id=id, name=name)
            team_owner=TeamOwner(value=team_owner),
        )

    # domain business_methods
```

The power of Value Objects lies in their ability to:

- Encapsulate logic more effectively.
- Provide greater expressiveness about the domain.
- Reduce clutter in domain entities.
- Improve testability.
- Increase maintainability, as they can be reused across different entities.
- Allow entities to focus on their identities and lifecycles.

:exclamation: `Bear in mind that not splitting your domain entity into different VOs initially is not necessarily a bad practice. It can be a good approach to first write your entity and then extract its concepts into VOs as it grows.`

### Aggregates

An aggregate represent a set of domain entities that can be treated as a single unit. Normally one of those entities is the root of the aggregate. It also handles the domain events that occurred during the lifecile (we'll talk about domain entities later on).

:white_check_mark: **Do this:**

```python
@dataclass(frozen=True)
class Team(AggregateRoot[UUID]):
    id: UUID
    name: str
    players: list[Player] = field(default_factory=list)

    # __post__init__

    @classmethod
    def new(cls, name: str, id: UUID = uuid4()) -> "Team":
        team = cls(
            name=name,
            id=id,
            players=list[Player]()
        )
        team.register_event(TeamCreated(entity=team))
        return team

    @classmethod
    def build(cls, name: str, players: list[Player], id: UUID) -> "Team":
        return cls(
            name=name,
            id=id,
            players=players
        )


    def hire_player(self, player: Player) -> "Team":
        if len(self.players) > 20:
            raise InvalidDomainError("Team can only have 20 players")

        player.hire(team=self.id)
        players = self.players
        players.append(player)

        team = self.clone(players=players)
        team.register_event(PlayerHired(entity=team))

        return team

    # validations methods

@dataclass(frozen=True)
class Player(Entity[UUID]):
    name: str
    id: UUID = field(default_factory=uuid4)
    team_owner: UUID = field(default=None)

    def __post_init__(self) -> None:
        self._validate_id()
        self._validate_name()

    @classmethod
    def new(cls, name: str, team_owner: UUID, id: UUID = uuid4()) -> "Player":
        return cls(
            name=name,
            id=id,
            team_owner=team_owner,
        )

    def is_free_agent(self) -> bool:
        return self.team_owner is None

    def hire(self, team: UUID) -> "Player":
        if not self.is_free_agent():
            raise InvalidDomainError("Player cannot be hired because it is a free agent")
        if self.team_owner == team:
            raise InvalidDomainError(f"Player is already hired by given team {team}")
        return self.clone(free_agent=False, team_owner=team)

    # validations methods
```

As you might notice, we now have another creational method, `build`, in addition to `new`, which we've seen so far. This distinction is important because creating a new entity and building an existing entity represent different states in their lifecycle.

Initially, a `Team` is created without players, as it exists independently of them. Subsequently, players are hired by the team. However, when you're building a team that has already been created, it receives a list of players that belong to that team. This distinction is why it makes sense to have two different methods for creating and building entities/aggregates. Additionally, the event that registers the creation of the `Team` occurs only once.

> :exclamation: Important rule:
>
> - `new`: Creates the bare minimum representitation of the entity\aggregate to exists and register it's creation within a domain event.
> - `build`: Creates the whole shape of an entity\aggregate that already exists.

### Repositories

A `Repository` is a design pattern that provides an abstraction layer for accessing and managing Aggregates and Entities. Repositories play a key role in the persistence and retrieval of domain objects, allowing domain logic to stay decoupled from data access code.

We have a `BaseRepository` abstraction:

```python
class BaseRepository(ABC, Generic[TEntity, TId]):
    @abstractmethod
    async def save(self, entity: TEntity) -> None:
        raise NotImplementedError()

    @abstractmethod
    async def find_all(self, filter: dict[str, any], limit: int = 20, skip: int = 0) -> list[TEntity]:
        raise NotImplementedError()

    @abstractmethod
    async def find_by_id(self, id: TId) -> TEntity:
        raise NotImplementedError()

    @abstractmethod
    async def delete_by_id(self, id: TId) -> None:
        raise NotImplementedError()
```

> :exclamation: Important rule:
>
> As you can observe, we're passing a filter parameter to `find_all` method. Never couple it to a specific database technology. Instead, it should filter based on the domain model attributes, neither the database model nor the database technology.
> The specific implementation of the filter needed to query the database should be done in the repository implementation.

As a general rules for writing effective repositories:

- Always have an abstraction of the repository in the `domain` layer and a corresponding concrete implementation in the `infrastructure` layer (Dependency Inversion Principle).
- Leverage on the `BaseRepository` abstraction.
- Never put domain logic inside repositories. They should remain agnostic to domain operations.
- Whenever possible, avoid implementing separate methods for updating and inserting. Instead, create a `save` method that will insert or update the entity based on its existence. Only consider separate methods like `add` and `update` if the complexity of a single `save` method is too high or if there are specific reasons, like database locking, that justify them.
- Decouple the database models from the domain models.
- Implement parsers (local factories or class factories) to translate between domain models and database models.
- Avoid raising exceptions that are framework related. Capture exception like `DbClient.NotFoundException` and raise our own `NotFoundException` instead.

:white_check_mark: **Do this:**

```python
class TeamRepository(ABC, BaseRepository[Team, UUID]):
    @abstractmethod
    async def find_by_name(self, name: str) -> Team
        raise NotImplementedError()


class MongoTeamRepository(TeamRepository):
    def __init__(self, parser: MongoTeamParser, database: "AgnosticDatabase[dict[str, Any]]"):
        self._parser = parser
        self._collection = database.get_collection("teams")

    async def save(self, entity: Team) -> None:
        team_document = self._parser.parse(entity)
        self._collection.replace_one(filter={"_id": team_document["_id"]}, replacement=team_document, upsert=True)

    # Rest of the implementations
```

> :exclamation: Important rule:
> You'll find an example of implementing parsers [here](#repositories-1)

:no_entry: **Don't do this:**

```python
class MongoTeamRepository(TeamRepository):
    def __init__(self, parser: MongoTeamParser, database: "AgnosticDatabase[dict[str, Any]]"):
        self._parser = parser
        self._collection = database.get_collection("teams")

    async def delete_player_from_team(self, player_id: UUID, team: Team) -> None
        players = [player for player in team.players if player.id != player_id]
        new_team = Team.build(
            id=team.id,
            name=team.name,
            players=players
        )

        self._collection.update_one({"id": new_team.id}, {"$set": self.__parser.to_database_object(new_team)})
```

Never put domain logic inside repositories, instead, move this logic to the domain entity itself. A repository should remain agnostic of domain operations.

:no_entry: **Don't do this:**

```python
class TeamRepository:
    def __init__(self, parser: MongoTeamParser, database: "AgnosticDatabase[dict[str, Any]]"):
        self._parser = parser
        self._collection = database.get_collection("teams")

    async def save(self, entity: Team) -> None:
        team_document = self._parser.parse(entity)
        self._collection.replace_one(filter={"_id": team_document["_id"]}, replacement=team_document, upsert=True)
```

This single implementation is couple to the database technology.

### Domain Services

Domain Services are classes that contain domain logic that doesn’t naturally fit within an Entity
or Value Object. They serve as a way to handle complex business operations that might span multiple
Aggregates or involve complex calculations and processes that don't belong in any one specific
domain model.

As a general rule for writing effective domain services:

- If the domain logic fits within the entity/aggregate, you don’t need to create a domain service.
- Splitting domain logic into multiple domain services can be a sign of an anemic domain.
- Only create a domain service when that logic doesn’t naturally fit within an entity or aggregate.
- Create an abstraction in the domain layer. Implement it in the domain layer if it doesn’t need to communicate
  with external systems or use external dependencies. Otherwise, implement it in the infrastructure layer.
  Most of the time, if the service only interacts with the domain and domain abstractions like repositories,
  you don’t need to create an abstraction—just a direct implementation will suffice.

:white_check_mark: **Do this:**

```python
class TeamStatisticsCalculator:
    def __init__(self, team_repository: TeamRepository, match_repository: MatchRepository):
        self._team_repository = team_repository
        self._match_repository = match_repository

    def calculate(self, team_id: UUID) -> "TeamStatistics":
        team = self._team_repository.find_by_id(team_id)
        if team is None:
            raise TeamNotFoundError()
        matches = self._match_repository.find_all_in_last_month(team_id=team_id)

        matches_won = [match for match in matches if match.is_won()]
        matches_lost = [match for match in matches if match.is_lost()]

        win_percentage = (len(matches_won) / len(matches)) * 100
        lost_percentage = (len(matches_lost) / len(matches)) * 100

        return TeamStatistic.create(win_percentage, lost_percentage)
```

:white_check_mark: **Do this:**

```python
class TeamStatisticsCalculator(ABC):
    @abstractmethod
    def calculate(team_id: UUID) -> "TeamStatistics":
        raise NotImplementedError()

class GoalServeTeamStatisticsCalculator(TeamStatisticsCalculator):
    def calculate(team_id: UUID) -> "TeamStatistics":
        # call to "GoalServe" API for getting the results of the last month
        # and calculate the statistic
```

In this case we need to interact with two different aggregates `Team` and `Match` to calculate the
statistics of a given team for the last month.

:no_entry: **Don't do this:**

```python
class TeamPlayerHiring:
    def __init__(self, team_repository: TeamRepository):
        self._team_repository = team_repository

    def calculate(self, team_id: UUID, player: Player) -> "Team":
        team = self._team_repository.find_by_id(team_id)
        if team is None:
            raise TeamNotFoundError()
        if len(team.players) > 20:
            raise InvalidDomainError("Team can only have 20 players")

        player.hire(team=team.id)
        players = team.players
        players.append(player)

        team = team.clone(players=players)
        team.register_event(PlayerHired(entity=team))

        return team
```

This logic belongs to the `Team` aggregate itself, there's no a real need of creating a domain
service for handling this. In fact, this way you're splitting the domain logic into many different
places and also mixing the responsibilities. In DDD, it’s generally best to keep domain-specific
logic within the aggregate that it pertains to, particularly when that logic directly concerns
the aggregate’s own data and operations. By keeping the hire_player method in Team, the aggregate
retains full control over its entities, making the design clearer and more cohesive.

### Domain Events

A **Domain Event** represents something significant that has happened within the domain. It signals an important state change
in a domain entity or aggregate, allowing different parts of the system to react to those changes asynchronously.

Domain events help maintain **loose coupling** between components by enabling event-driven communication, ensuring
that domain logic remains independent of side effects. Instead of calling other services or updating external systems directly,
an entity raises an event that is handled by other components as needed.

#### **Event-Driven Architecture in the Domain Layer**

To follow a clean event-driven approach, we use three core abstractions in our domain:

- **EventBus** – Publishes domain events.
- **EventSubscriber** – Subscribes to domain events.
- **EventHandler** – Processes domain events.

```python
class EventBus(ABC):
    @abstractmethod
    async def publish(self, domain_events: list[DomainEvent]) -> None:
        raise NotImplementedError()
```

```python
class EventSubscriber(ABC):
    @abstractmethod
    async def subscribe(self, events: type[SubscribedEvents]) -> None:
        raise NotImplementedError
```

```python
class EventHandler(ABC):
    @abstractmethod
    async def handle(self, event: IntegrationEvent) -> None:
        raise NotImplementedError
```

These abstractions allow us to keep our domain layer independent while enabling cross-domain communication.

#### **Domain Event Example**

A domain event encapsulates all relevant information about the change that occurred. Below is an example of a `DomainEvent` base class:

```python
@dataclass(frozen=True)
class DomainEvent(ABC):
    name: str
    domain: str
    entity: Entity
    payload: dict[str, Any]
    version: int = 1
    id: UUID = field(default_factory=uuid4)
    occurred_at: datetime = field(default=datetime.now(UTC))

    def to_dict(self) -> dict[str, Any]:
        return {
            "id": str(self.id),
            "domain": self.domain,
            "occurred_at": self.occurred_at.strftime('%Y-%m-%d %H:%M:%S'),
            "name": self.name,
            "entity": self.entity.dict(),
            "entityId": str(self.entity.id),
            "entityName": type(self.entity).__name__,
            "payload": self.payload,
            "version": self.version,
        }
```

📌 **Key Aspects of Domain Events:**

- **Immutable:** Events should not be modified after they are created.
- **Self-contained:** They contain all necessary information about what happened.
- **Timestamped:** Events are recorded with the exact time they occurred.
- **Serializable:** They can be easily stored or transmitted as structured data.

The **concrete implementations of event publishing and subscription** will be handled in the **Infrastructure Layer**, ensuring
that the domain remains independent of technical concerns. We will explore these implementations in the [Bus section](#bus) of this document.

## Application Layer

The **Application Layer** orchestrates the use of the domain model to implement the specific use cases of the application.
It coordinates user interactions, manages transactions, and ensures that the business rules defined
in the domain layer are correctly applied. This layer acts as an intermediary between the infrastructure layer and the domain layer,
ensuring clear separation of concerns.

Key Elements of the Application Layer:

- [Application Services](#application-services)
- [Commands and Responses](#command-and-responses)
- [Command Handlers and Query Handlers](#command-and-query-handlers)

### Application services

Manage business use cases and coordinate domain operations. We'll define an application service per use case.
They process input data encapsulated in a **Command** (we'll talk later about commands) DTO.

:white_check_mark: **Do this:**

```python
#Application Service
class ApplicationService(ABC):
    @abstractmethod
    def execute(self, command: Command) -> Response | None: raise NotImplementedError

class HirePlayer(ApplicationService):
    def __init__(self, team_repository: TeamRepository, event_bus: EventBus):
        self._team_repository = team_repository
        self._bus = event_bus

    def execute(self, command: HirePlayerCommand) -> None:
        team: Team = self.team_repository.find_by_id(command.team_id)
        if team is None:
            raise TeamNotFoundError('Team not found')

        player: Player = self.player_repository.find_by_id(command.player_id)

        if player is None:
            raise PlayerNotFoundError('Player not found')

        team.hire_player(player)

        self._team_repository.save(team)
        self._bus.publish(team.pull_events())
```

:no_entry: **Don't do this:**

```python
class HirePlayer(ApplicationService):
    def __init__(self, team_repository: TeamRepository, event_bus: EventBus):
        self._team_repository = team_repository
        self._bus = event_bus

    def execute(self, command: HirePlayerCommand) -> None:
        team: Team = self._team_repository.find_by_id(command.team_id)
        if team is None:
            raise TeamNotFoundError('Team not found')

        player: Player = self.player_repository.find_by_id(command.player_id)

        if player is None:
            raise PlayerNotFoundError('Player not found')

        if len(team.players) > 20:
            raise InvalidDomainError("Team can only have 20 players")

        player.hire(team=team.id)
        players = team.players
        players.append(player)

        team = team.clone(players=players)
        team.register_event(PlayerHired(entity=team))

        self._team_repository.save(team)
        self._bus.publish(team.pull_events())
```

Never place domain logic within an Application Service. Domain logic must be encapsulated within domain entities.
The role of an Application Service is strictly to orchestrate operations involving domain entities.

The same principle applies to external services: do not call external services directly from the Application Service. Instead,
rely on domain abstractions whose implementations reside in the infrastructure layer, such as repositories, event buses, clients...

As a general rule for writing effective application services:

- Application services should not contain domain logic. Instead, delegate the logic to methods
  encapsulated within the domain layer and focus on orchestrating domain operations.
- Choose expressive names for application services that clearly convey the purpose of the use case they implement.
  A well-named service makes the code easier to understand and maintain.
- Place the application service, along with its corresponding command and response objects, in the same folder.
  This structure keeps related components together and enhances code organization.

> :exclamation: An application service typically returns **None** when creating, modifying, or deleting data. It only returns
> a **Response** when retrieving data. However, you may return a **Response** in other cases if there is a justified reason to do so.

### Command And Responses

Commands and responses in the Application Layer are Data Transfer Objects (DTOs) that serve as input and output data carriers.
They ensure that the Application Service remains clean and focused on orchestration, while the data structures handle the information
flow between layers.

- **Commands** represent specific actions or requests made by a user or system.
- **Responses** encapsulate the result or outcome of executing an application service.

:white_check_mark: **Do this:**

```python
# Command DTO
@dataclass(frozen=True)
class Command:
    pass

@dataclass(frozen=True)
class HirePlayerCommand(Command):
    team_id: UUID
    player_id: UUID

# Response DTO
@dataclass(frozen=True)
class Response:
    pass

@dataclass(frozen=True)
class GetTeamResponse:
    name: str
    number_of_players: int
```

### Command and Query Handlers

**Command and Query Handlers** are concepts derived from **CQRS** (Command Query Responsibility Segregation),
a pattern that separates the responsibility for handling commands (writes) and queries (reads) in an application.
While CQRS can provide significant benefits in certain contexts, it's important to understand that it introduces
complexity and should only be used when necessary.

#### When Should You Use CQRS?

CQRS is particularly useful in scenarios where:

1. **Complex Business Logic for Reads and Writes**:
   When the logic for reading data and writing data is very different, CQRS allows you to optimize both
   operations independently. This can be useful in cases where:

   - The write operations are computationally expensive or involve complex domain logic.
   - The read operations require highly optimized data retrieval or have different requirements (e.g., read models or projections).

2. **Scalability Concerns**:
   If your application has high demands for either reads or writes and these demands are asymmetric, separating the handling of commands and
   queries can allow for scaling each operation independently. For example:

   - If your system is read-heavy (e.g., dashboards, search engines), you can scale your query handling without affecting the command side.
   - If writes are very frequent (e.g., in high-volume transaction systems), you can scale the write side separately.

3. **Event Sourcing**:
   In systems that use **Event Sourcing**, CQRS is often a natural fit because event sourcing emphasizes capturing state changes as events,
   which are more easily handled when commands (writes) and queries (reads) are separated. The write model captures events, and the read model
   is a projection of those events, often optimized for specific queries.

4. **Complex Permission or Security Models**:
   If the authorization requirements for reading and writing data are different, CQRS can help by separating the concerns of access control
   for commands (e.g., updates, deletions) versus queries (e.g., reads).

#### When Should You Avoid CQRS?

If your application does not face the challenges mentioned above, CQRS can introduce unnecessary complexity.
For simple applications where read and write logic is straightforward, using separate command and query handlers can add:

- **Overhead**: More code to maintain and understand, including multiple layers of handlers, buses, and extra infrastructure.
- **Increased Development Complexity**: More time spent setting up and managing the infrastructure for commands and queries,
  including additional considerations for consistency and data synchronization between the read and write models.

In most applications, a **single application service** that handles both commands and queries is sufficient and much easier to maintain.
By separating concerns logically within the service, you can manage domain logic and data flow without needing the additional complexity
of the CQRS pattern. Introducing CQRS should only be considered if there are clear, justified reasons, such as complex business logic,
high scalability requirements, or the use of event sourcing. For most use cases, the application service pattern provides a more straightforward
and maintainable solution.

#### Command Handlers

Command Handlers are responsible for executing specific commands in the system. They process input data encapsulated in a **Command**
DTO and delegate the execution to the appropriate **Application Service**. This separation ensures clear boundaries and responsibilities
between input handling and business logic orchestration.

Command Handlers typically:

1. **Receive a Command**: Contain the input data necessary for the operation.
2. **Validate the Command**: Ensure the input data is correct and complete.
3. **Delegate Execution**: Pass the command through a **Command bus** to the corresponding Application Service for processing.

:white_check_mark: **Do this:**

```python
# Command Definition
@dataclass(frozen=True)
class Command:
    pass

# Command Handler Definition
class CommandHandler(ABC):
    @abstractmethod
    async def handle(self, command: Command) -> Response | None:
        raise NotImplementedError

# Command DTO
@dataclass(frozen=True)
class CreateTeamCommand(Command):
    name: str


# Command Handler
class CreateTeamCommandHandler(CommandHandler):
    def __init__(self, create_team: CreateTeam):
        self._create_team = create_team

    async def handle(self, command: CreateTeamCommand) -> None:
        # Validate the command
        if not command.team_id or not command.player_id:
            raise ValueError("Invalid command: team_id and player_id are required")
        # Execute the application service associated to this handler
        await self._create_team.execute(command)


# Command Bus
class CommandBus:
    def __init__(self, command_handlers: dict[type[Command], Callable[..., CommandHandler]]):
        self._handlers = {}
        for command_type in command_handlers.keys():
            self.register_handler(command_type, command_handlers[command_type])

    def register_handler(self, command_type: type[Command], handler: [CommandHandler]):
        self._handlers[command_type] = handler

    async def dispatch(self, command: Command) -> Command | None:
        command_type = type(command)
        if command_type not in self._handlers:
            raise ValueError(f"No handler registered for command type: {command_type}")
        await self._handlers[command_type].handle(command)


# Application Service
class CreateTeam(ApplicationService):
    def __init__(self, repository: TeamRepository):
        self._repository = repository

    async def execute(self, command: CreateTeamCommand) -> None:
        team = Team.new(name=command.name)
        await self._repository.save(team)
```

> :exclamation: The command bus is typically called from an entry point, such as an API endpoint or an event subscriber.

#### Query Handlers

**Query Handlers** are responsible for processing queries and returning the requested data. Unlike commands,
queries are read-only operations, focusing solely on retrieving information without modifying the state of the system. Query handlers
are typically invoked through a **Query Bus**, which dispatches the query to the appropriate handler.

:white_check_mark: **Do this:**

```python
# Query DTO
@dataclass(frozen=True)
class GetTeamQuery(Query):
    team_id: UUID

# Response DTO
@dataclass(frozen=True)
class GetTeamResponse(Response):
    name: str
    number_of_players: int

# Query Bus
class QueryBus:
    def __init__(self, query_handlers: dict[type[Query], Callable[..., QueryHandler]]):
        self._handlers = {}
        for query_type in query_handlers.keys():
            self.register_handler(query_type, query_handlers[query_type])

    def register_handler(self, query_type: type[Query], handler: [QueryHandler]):
        self._handlers[query_type] = handler

    async def dispatch(self, query: Query) -> Response:
        query_type = type(query)
        if query_type not in self._handlers:
            raise ValueError(f"No handler registered for query type: {query_type}")
        return await self._handlers[query_type].handle(query)

# Query Handler
class GetTeamQueryHandler:
    def __init__(self, get_team: GetTeam):
        self._get_team = get_team

    async def handle(self, query: GetTeamQuery) -> GetTeamResponse:
        # Validate the query
        if not query.team_id:
            raise ValueError("Invalid query: team_id is required")

        # Execute the application service associated to this handler
        return await self._get_team.execute(query)

# Application Service
class GetTeam:
    def __init__(self, repository: TeamRepository):
        self._repository = repository

    async def execute(self, query: GetTeamQuery) -> GetTeamResponse:
        team = await self._repository.find_by_id(query.team_id)
        if team is None:
            raise TeamNotFoundError("Team not found")

        return GetTeamResponse(name=team.name, number_of_players=len(team.players))
```

> :exclamation: The query bus is typically called from an entry point, such as an API endpoint or an event subscriber.

## Infrastructure Layer

The **Infrastructure Layer** supports the application by providing implementations for technical concerns and connecting the application to external systems. This layer ensures that the application’s core logic remains clean and focused on the business by abstracting low-level technical details.

Key Elements of the Infrastructure Layer:

- [Frameworks and Libraries](#frameworks-and-libraries)
- [API](#api)
- [Repositories](#repositories)
- [Bus](#bus)
- [Clients](#clients)
- [Services](#services)

The Infrastructure Layer bridges the gap between the domain/application layers and the external world, ensuring separation of concerns and clean code boundaries.

### Frameworks and Libraries

Provide reusable components and tools that simplify infrastructure concerns, such as logging, bootstrap config, http layer, dependency injection... At landbot we use FastAPI.

:white_check_mark: **Do this:**

```python
# main.py
logger = logging.getLogger(__name__) #Logger configuration
app = FastAPI()
add_error_handlers(app) #Global error handling

# Dependency injection container
container = Container()
container.init_resources()

# Exposed routes
app.include_router(healthcheck.router)
app.include_router(container.team_controller().routes())
```

### API

This is typically the main entry point of your application. It implements the HTTP layer or other communication protocols like gRPC, exposing application functionality to clients. We use FastAPI for creating our endpoints.

:white_check_mark: **Do this:**

```python
# BaseController
class BaseController(ABC):
    @abstractmethod
    def __init__(self):
        pass

    @abstractmethod
    def _register_routes(self) -> None:
        pass

    @abstractmethod
    def routes(self) -> Any:
        pass

router = APIRouter(
    prefix="/api/v1/teams",
    tags=["Teams"],
    responses={
        status.HTTP_500_INTERNAL_SERVER_ERROR: {
            "description": "Internal server error.",
            "model": Error,
        },
        status.HTTP_503_SERVICE_UNAVAILABLE: {
            "description": "The server is currently unable to handle the request.",
            "model": Error,
        },
    },
)

# Without Event Bus
class TeamController(BaseController):
    def __init__(self, create_team: CreateTeam):
        self._router = router
        self.register_routes()
        self._create_team = create_team

    def routes(self) -> APIRouter:
        return self._router

    def register_routes(self) -> None:
        self._router.add_api_route("", self.create_team, methods=["POST"], name="Create a new team")

    async def create_team(self, request: CreateCharacterRequest) -> Response:
        command = CreateTeamCommand(name=request.name)

        response = await self._create_team.execute(command)

        return to_response(status_code=status.HTTP_201_CREATED, model=response)

# With Event Bus
class TeamController(BaseController):
    def __init__(self, command_bus: CommandBus, query_bus: QueryBus):
        self._router = router
        self._register_routes()
        self._command_bus = command_bus
        self._query_bus = query_bus

    def routes(self) -> APIRouter:
        return self._router

    def _register_routes(self) -> None:
        self._router.add_api_route("", self.create_team, methods=["POST"], name="Create a new team")
        self._router.add_api_route("/{team_id}", self.get_team, methods=["GET"], name="Get given team")

    async def create_team(self, request: CreateTeamRequest) -> Response:
        command = CreateTeamCommand(name=request.name)

        await self._command_bus.dispatch(command)

        return to_response(status_code=status.HTTP_201_CREATED)
```

Implementing a class controller give us the following advantages:

- Dependencies, like services (`CreateTeam` in this example), can be injected once into the controller's constructor. This reduces the need for repetitive dependency declarations for each endpoint and provides a cleaner way to manage and extend dependencies.
- Centralizing route registration within a class ensures consistency and simplifies route management. The `register_routes` method acts as a single place to define all the endpoints handled by the controller, improving readability and organization.
- A class-based design aligns well with object-oriented principles, promoting encapsulation, modularity, and abstraction.
- Testing a class-based controller allows for better isolation. Mocking or injecting dependencies is straightforward, and all related endpoints can be tested as part of a single unit. This reduces the boilerplate required to set up test environments for each endpoint individually.

### Repositories

They are concrete implementations of repository interfaces defined in the domain layer, responsible for interacting with the database or other storage systems.

:white_check_mark: **Do this:**

```python
class MongoTeamRepository(TeamRepository):
    def __init__(self, parser: MongoTeamParser, database: "AgnosticDatabase[dict[str, Any]]"):
        self._parser = parser
        self._collection = database.get_collection("teams")

    async def save(self, entity: Team) -> None:
        team_document = self._parser.parse(entity)
        self._collection.replace_one(filter={"_id": team_document["_id"]}, replacement=team_document, upsert=True)

    # Rest of the implementations
```

Implement parsers like this:

```python
class DatabaseParser(ABC):
    @abstractmethod
    def to_database_object(self, domain: Entity) -> dict:
        raise NotImplementedError

    @abstractmethod
    def to_domain_object(self, database_object: dict) -> Entity:
        raise NotImplementedError

class MongoTeamParser(DatabaseParser):
    def __init__(self, player_parser: PlayerTeamParser):
        self._player_parser = player_parser

    def to_database_object(self, domain: Team) -> dict:
        return {
            "_id": domain.id,
            "name": domain.name,
            "players": [self.__player_parser.to_database_object(player) for player in domain.players]
        }

    def to_domain_object(self, database_object: dict) -> Entity:
        return Team.build(
            id=uuid.UUID(database_object["_id"]),
            name=database_object["name"],
            players=[self.__player_parser.to_domain_object(player) for player in database_object["players"]]
        )

class PlayerTeamParser(DatabaseParser):
    def to_database_object(self, domain: Player) -> dict:
        return {
            "_id": domain.id,
            "name": domain.name,
            "team_owner": domain.team_owner,
        }
    def to_domain_object(self, database_object: dict) -> Entity:
        return Player.build(
            id=uuid.UUID(database_object["_id"]),
            name=database_object["name"],
            team_owner=database_object["team_owner"]
        )
```

Follow the same rules we explained in the [domain layer](#repositories) when we talked about the repositories.

### Bus

The **Bus** is responsible for handling the **dispatch and processing of events**, enabling asynchronous
communication between different parts of the system or external services. It acts as a key component in
event-driven architectures, ensuring that events are published, consumed, and processed efficiently.

At Landbot, we utilize a **Google PubSub** to implement the bus, providing a decoupled way to handle
cross-domain interactions. The bus is primarily composed of two main components:

- **EventBus** – Handles event publishing.
- **EventSubscriber** – Listens for and processes incoming events.

#### **Integration Events vs. Domain Events**

- **Domain Events** occur within a domain and are used for internal communication, maintaining consistency within the domain.
- **Integration Events** are used to communicate across different domains or services, ensuring proper decoupling between bounded contexts.

```python
@dataclass(frozen=True)
class IntegrationEvent:
    name: str
    payload: dict[str, Any]
    version: int

    @classmethod
    def new(cls, name: str, payload: dict[str, Any], version: int) -> 'IntegrationEvent':
        return cls(name, payload, version)

    def validate(self) -> None:
        if not self.name or not isinstance(self.name, str):
            raise ValueError("Integration event name must be a non-empty string")
        if not isinstance(self.version, int) or self.version < 1:
            raise ValueError("Integration event version must be a positive integer")
        if not isinstance(self.payload, dict):
            raise ValueError("Integration event payload must be a dictionary")
```

With this implementation, our system efficiently processes **both domain and integration events**, ensuring reliability and scalability.

#### **PubSub Event Bus Implementation**

```python
    async def publish(self, domain_events: list[DomainEvent]) -> None:
        if not domain_events:
            return None

        for event in domain_events:
            try:
                topic_name = (
                    f"{event.domain}-{event.name}-{os.getenv('EPHEMERAL_SUBDOMAIN')}"
                    if os.getenv("EPHEMERAL_SUBDOMAIN")
                    else f"{event.domain}-{event.name}"
                )
                topic = self._client.topic_path(project=self._project_id, topic=topic_name)
                future = self._client.publish(topic=topic, data=json.dumps(event.to_dict()).encode("utf-8"))
                future.result()
            except Exception as e:
                self._logger.exception(f'There was an error publishing event due to: {e}')
```

> :exclamation: This `f"{event.domain}-{event.name}-{os.getenv('EPHEMERAL_SUBDOMAIN')}"` is a workaround to get it working in our ephemeral env
> since we're creating the topics dynamically for each branch.

#### **PubSub Event Subscriber Implementation**

```python
class PubSubEventSubscriber(EventSubscriber):
    def __init__(self, client: pubsub_v1.SubscriberClient, subscription: str, event_handler: EventHandler, logger: logging.Logger):
        self._client = client
        self._subscription = subscription
        self._event_handler = event_handler
        self._logger = logger

    async def subscribe(self) -> None:
        def callback(message: Message) -> None:
            event_data = json.loads(message.data.decode("utf-8"))
            event = IntegrationEvent(**event_data)
            asyncio.create_task(self._event_handler.handle(event))
            message.ack()

        self._client.subscribe(self._subscription, callback=callback)
        self._logger.info(f'Subscribed to {self._subscription}')
```

You can find the full implementation at this [ADR](https://git.yexir.com/landbot/architecture/-/blob/main/ADR/backend/0004-topics-architecture.md?ref_type=heads). We also have the sync implementation of this [here](https://git.yexir.com/landbot/service-scaffolding/-/tree/main/src/shared/infrastructure/bus/events?ref_type=heads) if you have to use them along with Django.

### Clients

The **Clients** section covers how external systems or internal services interact with our system through well-defined APIs and protocols. Clients are responsible for making requests and processing responses, ensuring seamless communication between different services.

At Landbot, we ensure that our clients interact with services efficiently and reliably, using best practices such as **dependency injection** and the **anticorruption layer pattern** to maintain system integrity.
By structuring our client interactions using **HttpClient** and the **Anticorruption Layer**, we ensure
that our system remains **robust, adaptable, and protected from external system changes**.

#### **HTTP Client Implementation**

One way clients interact with services is via HTTP requests. Below is an example of a Python HTTP client using `httpx` for making asynchronous API calls:

```python
import httpx

class HttpClient:
    def __init__(self, base_url: str, client: httpx.AsyncClient):
        self._base_url = base_url
        self._client = client

    async def get(self, endpoint: str, params: dict = None) -> dict:
            response = await self._client.get(f"{self._base_url}{endpoint}", params=params)
            response.raise_for_status()
            return response.json()

    async def post(self, endpoint: str, data: dict) -> dict:
            response = await self._client.post(f"{self._base_url}{endpoint}", json=data)
            response.raise_for_status()
            return response.json()

    # Rest of the HTTP methods needed
```

#### **Using the Anticorruption Layer Pattern**

The **Anticorruption Layer (ACL)** pattern acts as an intermediary between our internal system
and external APIs, ensuring that we control how external data is processed and mapped to our domain
model. By injecting the `HttpClient` into the ACL, we maintain a clean separation between external APIs
and our business logic.

:white_check_mark: **Do this:**

```python
class LaLigaAPIClient:
    def __init__(self, http_client: HttpClient, api_url: str):
        self._http_client = http_client
        self._api_url = api_url

    async def fetch_team(self, team_id: str) -> Team:
        raw_data = await self._http_client.get(f"{self._api_url}/teams/{team_id}")
        return self._map_to_team(raw_data)

    # Rest of the API methods

    def _map_to_team(self, raw_data: dict) -> Team:
        return Team.build(
            id=UUID(raw_data["id"]),
            name=raw_data["name"],
            players=[]
        )
```

> :exclamation: The responsibility of an Anticorruption Layer is to **transform external data** before using it internally.
> Never place domain logic within the ACL. Instead, focus on mapping external data to domain entities and ensuring data integrity.

📌 **Key Benefits of the Anticorruption Layer:**

- **Decouples our domain model from external APIs**, preventing external changes from affecting internal logic.
- **Ensures controlled transformation and validation** of external data before using it internally.
- **Improves maintainability** by centralizing API interactions in a single, well-defined interface.

### Services

Services in the **Infrastructure Layer** facilitate communication between the application and external systems, such as authentication, monitoring, and observability.
These services are also known as **adapters** in Clean/Hexagonal Architecture, as they provide a bridge between domain logic and external dependencies.

#### Why Implement These Services in the Infrastructure Layer?

Unlike **domain services**, which encapsulate domain logic that does not naturally fit into an entity or value object, **infrastructure services** provide functionality
that depends on external libraries, APIs, or systems. This distinction is essential for maintaining a clean architecture and ensuring that domain logic remains independent of external concerns.

#### **Key Differences Between Domain Services and Infrastructure Services**

| Feature      | Domain Service                                                           | Infrastructure Service                                               |
| ------------ | ------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| Purpose      | Encapsulates domain logic that doesn’t belong in an entity or aggregate. | Provides technical capabilities that interact with external systems. |
| Dependencies | Pure domain logic, no external dependencies.                             | Relies on third-party libraries, APIs, or external systems.          |
| Placement    | Resides in the **Domain Layer**.                                         | Resides in the **Infrastructure Layer**.                             |
| Example      | Business rules like calculating statistics.                              | Token verification, sending emails, logging...                       |

#### Implementing an Infrastructure Service

A common example of an external service is an **access token verifier** that checks the validity of JWT tokens. Since this functionality relies on an external cryptographic library,
the implementation resides in the **Infrastructure Layer**, while the abstraction belongs to the **Domain Layer**.

```python
from abc import ABC, abstractmethod

class AccessTokenVerifier(ABC):
    """Abstraction for verifying access tokens. Resides in the Domain Layer."""

    @abstractmethod
    def verify(self, token: str) -> dict:
        """Verifies the given token and returns the decoded payload if valid."""
        raise NotImplementedError()
```

```python
import jwt
from jwt import PyJWTError
from typing import Any, Dict
from domain.access_token_verifier import AccessTokenVerifier

class JwtAccessTokenVerifier(AccessTokenVerifier):
    """Concrete implementation of AccessTokenVerifier using JWT. Resides in the Infrastructure Layer."""

    def __init__(self, secret_key: str, algorithms: list[str]):
        self._secret_key = secret_key
        self._algorithms = algorithms

    def verify(self, token: str) -> Dict[str, Any]:
        try:
            decoded_token = jwt.decode(token, self._secret_key, algorithms=self._algorithms)
            return decoded_token
        except PyJWTError as e:
            raise ValueError(f"Invalid token: {str(e)}")
```

:white_check_mark: **Do this:** Implement the verifier in the **Infrastructure Layer** because it relies on an external cryptographic library (`PyJWT`).

:no_entry: **Don't do this:** Implement token verification directly inside a **Domain Service**, as it would introduce an external dependency in the domain layer, violating
clean architecture principles.

📌 **Key Takeaways**

- **Domain Services vs. Infrastructure Services**: Domain services encapsulate domain logic, while infrastructure services handle external interactions.
- **Dependency Inversion Principle**: Keep abstractions in the **Domain Layer** while implementing them in the **Infrastructure Layer** when they require external dependencies.
- **Clean Architecture Compliance**: Ensures a **clear separation of concerns**, making the codebase more **maintainable and flexible**.

By following these principles, we ensure that our software remains modular, testable, and easy to evolve over time.
