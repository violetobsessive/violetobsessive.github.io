---
author: Frances Fu
pubDatetime: 2026-02-08T21:40:00Z
modDatetime: 2025-02-08T00:16:00Z
title: TDD, BDD and DDD in Software Development
slug: testing approaches
featured: true
draft: false
tags:
  - Testing
  - C#
description: They are different approaches in software development that emphasize testing and collaboration. I'll explain the differences in this post with concrete C# code examples.
---

## Table of contents

## Why Does these methodologies Exist?

In today's software development pipeline, it can repeatedly runs into the same problems:

- Requirements ambiguity: disagreement about business rules until late.
- Infrastructure decoupling: Code becomes hard to change without breaking things.

That's where these practices come in — they response to these recurring challenges in everyday work. They each optimizes a different part of the system —— building safely (TDD), agreeing on behavior (BDD), and structuring business logic (DDD).

## How TDD works

`Test-Driven Development (TDD)` typically means that automated tests get written before the actual code. You start by adding a test that fails, then implementing just enough code to make that test pass, before doing the final clean up and refactor.

This tight loop gives you constant feedback, makes bugs easier to catch early, and mostly importantly, it makes your changes more flexible, and allow a cleaner design.

Here's an example of TDD.

Imagine we have a bank account that is retrieved and stored through `IBankAccountRepository`, and that exposes a `depositToAccont` method;
`BankAccountService` passes in `id`, `amount`, calls depositToAccount method, and then persists the updated account using the repository.

```csharp
public interface IBankAccountRepository
{
    void Store(BankAccount account);
    BankAccount Get(int id);
}
```

```csharp
public class BankAccountService(IBankAccountRepository _repository)
{
    public void DepositToAccount(int id, decimal amount)
    {
        var account = _repository.Get(id);
        account.Deposit(amount);
        _repository.Store(account);
    }
}

```

Here's a test-driven development approach in NSubstitute:

```csharp
     [Fact]
    public void Using_mocks()
    {
        // Arrange
        var repo = Substitute.For<IBankAccountRepository>();
        var existingAccount = Substitute.For<BankAccount>(123, 0m);

        repo.Get(123).Returns(existingAccount);

        var sut = new BankAccountService(repo);

        // Act
        sut.DepositToAccount(123, 100m);

        // Assert
        repo.Received().Get(123);
        existingAccount.Received().Deposit(100m);
        repo.Received().Store(existingAccount);
    }
```

Before implementation, we want to drive the beahaviour of `depositToAccont`, making sure it

- gets the account
- deposit the specified amount
- store the correct amount

In order to do that, we mock the repository, create a substitute for bank account to expose the method/service logic under test.

The test focuses on one behavior —— deposit money into an account via `BankAccountService`. This approaches drive clearer structure, and facilitates the safety of refactoring.

## How BDD works

`Behaviour-Driven Development (BDD)` evolved from TDD but focuses on business value and user behavior rather than implementation details. It begins by clarifying desired behavior the team wants the code to behave.

Then we describe that behaviour in **Gherkin** using **Given–When–Then** statements. It serve as a guide for implementing code that satisfies the specified outcomes. This is a more common approaches in corporate/enterprise settings.

Here's a behaviour-driven development approach in NSubstitute with the same example:

```csharp
  [Fact]
    public void Using_mocks()
    {
        // Given
        var repo = Substitute.For<IBankAccountRepository>();
        var existingAccount = Substitute.For<BankAccount>(123, 0m);

        repo.Get(123).Returns(existingAccount);

        var service = new BankAccountService(repo);

        // When
        service.DepositToAccount(123, 50m);

        // Then
        existingAccount.Received().Deposit(50m);
        repo.Received().Store(existingAccount);
    }
```

In this example, Given–When–Then is as follow -

Given an existing bank account
When I deposit money
Then the account is updated and persisted.

More specifically,
**Given**:

- repo and existingAccount are created via NSubstitute.
- repo.Get(123).Returns(existingAccount); sets up the context: “There is an account with ID 123.”

**When**:

- service.DepositToAccount(123, 50m);
- This is the event/trigger in the scenario: “When depositing 50 into account 123.”

**Then**:

- existingAccount.Received().Deposit(50m);
  - the account should receive a deposit of 50.
- repo.Received().Store(existingAccount);
  - updated account should be persisted.

This approach is mostly beneficial for readablity and clear structure of requirements, commonly used to create JIRA tickets. It's still interaction-based, but phrased in business/behavior terms rather than just "calls this method."

## How DDD works

`Domain-Driven Design (DDD)` focuses on modeling software to match the business domain. As systems grow, the hardest part often becomes the rules of the business, not the tech.

In complex domains (banking, insurance, logistics, marketplaces), rules are numerous, evolving, full of exceptions, and deeply interconnected. Without a domain-centered design, logic ends up scattered i.e. validation in controllers, calculations in services.

DDD emphasizes collaboration between technical and domain experts to create a shared understanding and language around business concepts.

Here's a domain-driven development approach in NSubstitute with the same example:

```csharp
[Fact]
    public void DepositToAccount_Represents_A_Money_Inflow_For_The_BankAccount_Aggregate()
    {
        // Arrange (Application service context)
        var repo = Substitute.For<IBankAccountRepository>();
        var bankAccountAggregate = Substitute.For<BankAccount>(123, 200m); // existing balance

        repo.Get(123).Returns(bankAccountAggregate);

        var appService = new BankAccountService(repo);

        // Act (Application command)
        appService.DepositToAccount(123, 80m);

        // Assert (Domain orchestration expectations)
        bankAccountAggregate.Received().Deposit(80m);
        repo.Received().Store(bankAccountAggregate);
    }
```

DDD treats `BankAccountService` as an **application service** and `BankAccount` as an **aggregate root**. The test verifies that a command (`DepositToAccount`) results in the correct domain operation and persistence. Essentially:

- BankAccountService = application service.
- IBankAccountRepository = domain repository abstraction.
- BankAccount = aggregate root (we’re mocking it just to verify orchestration).

The test is making sure that `DepositToAccount` command orchestrates the right steps in the domain model —— it must load the aggregate it’s going to modify. As an application service should not directly manipulate raw data; it should:

- Load the aggregate through its repository.
- Let the aggregate enforce its own invariants (rules).
