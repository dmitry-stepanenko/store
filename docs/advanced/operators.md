# State Operators

## Why?

The NGXS `patchState` method is used to do [immutable object](https://en.wikipedia.org/wiki/Immutable_object) updates to the container state slice without the typical long-handed syntax. This is very neat and convenient because you do not have to use the `getState` and `setState` as well as the `Object.assign(...)`or the spread operator to update the state. The `patchState` method only offers a shallow patch and as a result is left wanting in more advanced scenarios. This is where state operators come in. The `setState` method can be passed a state operator which will be used to determine the new state.

## Basic

The basic idea of operators is that we could describe the modifications to the state using curried functions that are given any inputs that they need to describe the change and are finalized using the state slice that they are assigned to.

# Example

From theory to practice - let's take the following example:

```ts
import { Injectable } from '@angular/core';
import { State, Action, StateContext } from '@ngxs/store';
import { patch } from '@ngxs/store/operators';

export interface AnimalsStateModel {
  zebras: string[];
  pandas: string[];
  monkeys?: string[];
}

export class CreateMonkeys {
  static readonly type = '[Animals] Create monkeys';
}

@State<AnimalsStateModel>({
  name: 'animals',
  defaults: {
    zebras: [],
    pandas: []
  }
})
@Injectable()
export class AnimalsState {
  @Action(CreateMonkeys)
  createMonkeys(ctx: StateContext<AnimalsStateModel>) {
    ctx.setState(
      patch<AnimalsStateModel>({
        monkeys: []
      })
    );
  }
}
```

The `patch` operator expresses the intended modification quite nicely and returns a function that will apply these modifications as a new object based on the provided state.
In order to understand what this is doing let's express this in a long handed form:

```ts
  // For demonstration purposes! This long handed form is not needed from NGXS v3.4 onwards.
  @Action(CreateMonkeys)
  createMonkeys(ctx: StateContext<AnimalsStateModel>) {
    const state = ctx.getState();
    ctx.setState({
      ...state,
      monkeys: []
    });
  }
```

## Supplied State Operators

This is not the only operator, we introduce much more that can be used along with or in place of `patch`.

If you want to update the value of a property based on some condition - you can use `iif`, it's signature is:

```ts
iif<T>(
  condition: Predicate<T> | boolean,
  trueOperatorOrValue: StateOperator<T> | T,
  elseOperatorOrValue?: StateOperator<T> | T
): StateOperator<T>
```

If you want to update an item in the array using an operator or value - you can use `updateItem`, it's signature is:

```ts
updateItem<T>(selector: number | Predicate<T>, operator: T | StateOperator<T>): StateOperator<T[]>
```

If you want to remove an item from an array by index or predicate - you can use `removeItem`:

```ts
removeItem<T>(selector: number | Predicate<T>): StateOperator<T[]>
```

If you want to insert an item to an array, optionally before a specified index - use `insertItem` operator:

```ts
insertItem<T>(value: T, beforePosition?: number): StateOperator<T[]>
```

If you want to append specified items to the end of an array - the `append` operator is suitable for that:

```ts
append<T>(items: T[]): StateOperator<T[]>
```

It's also possible to compose multiple operators into a single operator that would apply each consecutively using `compose`:

```ts
compose<T>(...operators: StateOperator<T>[]): StateOperator<T>
```

These operators introduce a new way of declarative state mutation.

## Advanced Example

Let's look at more advanced examples:

```ts
import { Injectable } from '@angular/core';
import { State, Action, StateContext } from '@ngxs/store';
import { patch, append, removeItem, insertItem, updateItem } from '@ngxs/store/operators';

export interface AnimalsStateModel {
  zebras: string[];
  pandas: string[];
}

export class AddZebra {
  static readonly type = '[Animals] Add zebra';
  constructor(public payload: string) {}
}

export class RemovePanda {
  static readonly type = '[Animals] Remove panda';
  constructor(public payload: string) {}
}

export class ChangePandaName {
  static readonly type = '[Animals] Change panda name';
  constructor(public payload: { name: string; newName: string }) {}
}

@State<AnimalsStateModel>({
  name: 'animals',
  defaults: {
    zebras: ['Jimmy', 'Jake', 'Alan'],
    pandas: ['Michael', 'John']
  }
})
@Injectable()
export class AnimalsState {
  @Action(AddZebra)
  addZebra(ctx: StateContext<AnimalsStateModel>, action: AddZebra) {
    ctx.setState(
      patch<AnimalsStateModel>({
        zebras: append<string>([action.payload])
      })
    );
  }

  @Action(RemovePanda)
  removePanda(ctx: StateContext<AnimalsStateModel>, action: RemovePanda) {
    ctx.setState(
      patch<AnimalsStateModel>({
        pandas: removeItem<string>(name => name === action.payload)
      })
    );
  }

  @Action(ChangePandaName)
  changePandaName(ctx: StateContext<AnimalsStateModel>, action: ChangePandaName) {
    ctx.setState(
      patch<AnimalsStateModel>({
        pandas: updateItem<string>(
          name => name === action.payload.name,
          action.payload.newName
        )
      })
    );
  }
}
```

You will see that in each case above the state operators are wrapped within a call to the `patch` operator. This is only done because of the convenience that the `patch` state operator provides for targeting a nested property of the state.

## Typing Operators

Specifying types for the `patch` operator is always necessary when doing nested updates. You can face cases when the `patch` operator cannot infer the nested type structure. Let's look at the following state:

```ts
export class UpdateLine1 {
  static readonly type = '[Address] Update line1';
  constructor(readonly line1: string) {}
}

export interface AddressStateModel {
  country: {
    city: {
      address: {
        line1: string;
      };
    };
  };
}

@State<AddressStateModel>({
  name: 'address',
  defaults: {
    country: {
      city: {
        address: {
          line1: ''
        }
      }
    }
  }
})
@Injectable()
export class AddressState {
  @Action(UpdateLine1)
  updateLine1(ctx: StateContext<AddressStateModel>, action: UpdateLine1) {
    ctx.setState(
      patch({
        country: patch({
          city: patch({
            address: patch({
              line1: action.line1
            })
          })
        })
      })
    );
  }
}
```

If we don't specify the type explicitly for `patch`, all objects are inferred as `unknown`, meaning that TypeScript cannot tell us that we're doing something wrong or using the wrong type. The correct way of specifying nested types is shown below:

```ts
export class UserState {
  @Action(UpdateLine1)
  updateLine1(ctx: StateContext<AddressStateModel>, action: UpdateLine1) {
    ctx.setState(
      patch<AddressStateModel>({
        country: patch<AddressStateModel['country']>({
          city: patch<AddressStateModel['country']['city']>({
            address: patch<AddressStateModel['country']['city']['address']>({
              line1: action.line1
            })
          })
        })
      })
    );
  }
}
```

If we change `country` to `Qcountry` (intentional mistake), the compiler will tell us `Object literal may only specify known properties, but 'Qcountry' does not exist`. The same technique may be used with other operators if they cannot infer the type.

💡 Tip: we can specify the state model type and chain properties to get the desired type. Like in the example above.

## Custom Operators

You can also define your own operators for updates that are common to your domain. For example:

```ts
function addEntity(entity: Entity): StateOperator<EntitiesStateModel> {
  return (state: Readonly<EntitiesStateModel>) => {
    return {
      ...state,
      entities: { ...state.entities, [entity.id]: entity },
      ids: [...state.ids, entity.id]
    };
  };
}

interface CitiesStateModel {
  // ...
}

@State<CitiesStateModel>({
  name: 'cities',
  defaults: {
    entities: {},
    ids: []
  }
})
@Injectable()
export class CitiesState {
  @Action(AddCity)
  addCity(ctx: StateContext<CitiesStateModel>, action: AddCity) {
    ctx.setState(addEntity<CitiesStateModel>(action.payload.city));
  }
}
```

Here you can see that the developer chose to define a convenience method called `addEntity` for doing a common state modification. This operator could also have also been defined using existing operators like so:

```ts
function addEntity(entity: Entity): StateOperator<EntitiesStateModel> {
  return patch<EntitiesStateModel>({
    entities: patch({ [entity.id]: entity }),
    ids: append([entity.id])
  });
}
```

As you can see, state operators are very powerful to start moving your immutable state updates to be more declarative and expressive. Enhancing the overall maintainability and readability of your state class code.

## Snippets

Check this [section](./../snippets/operators.md) for more operators that you can add to your application.

## Relevant Articles

[NGXS State Operators](https://medium.com/ngxs/ngxs-state-operators-8b339641b220)
