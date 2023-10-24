# 🔥 Implementation of a Limit Order System on a Uniswap V3-Like Protocol

## 🌐 Context

This implementation was part of a hiring test, and although I had limited knowledge of Uniswap V3, the resulting solution is fairly intricate, potentially containing some issues related to the protocol.

It must be done initially based on Uniswap V3 code, but I asked to do it in Cairo so the challenge has been done on [YAS](https://github.com/lambdaclass/yet-another-swap) developed by [LamdaClass](https://github.com/lambdaclass) and its community.

The code base is not yet finished so some feature are missing such as the burn position.

Hopefully there is a PR but not yet finished, but I could grap some code from there to make it works, however I didn't build the burn mechanism since it was not the purpose of this challenge.

## 📝 Description

The objective was to devise a gas-efficient Limit Order system that doesn't rely on an off-chain mechanism.

## 💡 Theory

Orders are user-specific, and executing an order depends on a price event (in our case, when a specific price is reached). There are essentially two theoretical solutions to manage this:
- Execute the order when the event occurs.
- Record the event somehow to retroactively execute the order later.

The first solution is impractical, as all orders are user-customized, and executing the order during a user swap would be gas-inefficient.

Regarding the second solution, there are several possibilities that will be explored in the following sections.

### 📈 Tick Curves

Store the timestamp when an order is created.

Keep track of the lowest and highest ticks during swaps, along with their respective block timestamps. By doing this, a tick curve is created over time. When a user fulfills their order, they will analyze the entire tick curve (considering highest or lowest ticks based on whether the order was 0for1 or 1for0) to determine if their tick has been crossed.

This solution has limitations, with the primary one being that the curve size will expand, resulting in increased gas consumption when fulfilling orders, even if Starknet computation fees are low.

### 👥 Subgroup of Orders

My initial attempt involved creating a hash map to store orders by tick. Swappers would then iterate over these orders to identify and fulfill them. While this method ensures orders are processed in a first-come-first-serve (FCFS) order, it is less gas-efficient than the previous solution. However, it is likely the fairest approach.

### 🚩 Flagging all Orders once per Tick

The implemented solution involves flagging all orders once per tick. Further details on this approach will be provided in the subsequent description.

## 👷 Implementation

### Fill order mechanism

<img width="800" alt="image" src="https://github.com/Bal7hazar/yet-another-swap/assets/97087040/1eb4140b-2f45-4a6f-88e9-a3a529625a6a">

When a swap occurs, if the tick changes the the swapper will then automatically flag all corresponding orders.

It is done with only one storage change for a single tick, from a value to 0, so it won't involve a high gas price.

To understand how is it possible we need to develop the storage architecture of the orders.

### Order storage

<img width="800" alt="image" src="https://github.com/Bal7hazar/yet-another-swap/assets/97087040/c5f12aa8-83d7-4a37-b395-c2e77fbaabea">

The storage is composed of 4 hashmaps:

- The first one is simply a map of Orders, it will be used by the Orderer to collect its order:

  * OrderKey -> Order

- The second one is more complex and is in fact 3 hashmaps (required to store efficiently arrays) that store keys by Tick and by ZeroToOne status:

  * (Tick, ZeroToOne) -> Array Len of the OrderKeys
  * (Tick, ZeroToOne, index) -> OrderKey
  * (Tick, ZeroToOne, OrderKey) -> Index

So basically when a swap occurs and changes the current tick, the contract remove all arrays at this tick and in the direction `zero_to_one` by replacing the Array Len by 0. And that's it for the swapper part.

Note that new orders with the same tick and `zero_to_one` will erase the previous values in the 2 other hashmaps when they will be created.
  
### Create an Order
  
<img width="800" alt="image" src="https://github.com/Bal7hazar/yet-another-swap/assets/97087040/20588a8b-d46a-49a5-9843-9466c08d1f34">

When an order is created some checks are verified:

- The tick does not cross the current price
- The Order not already exists or must be collected

To do this we check first that the OrderKey stored at the OrderKey index is not the same that the one computed using inputs.

You may have notice an edge case, indeed it is possible to have the exact same OrderKey from a previous filled Order (since these maps are not eraased), therefore to ensure that the order doesn't exist we check for the Order map and see if the Order has been collected, it is not the case we throw saying the previous Order must be collected, otherwise we can safely replace the old one.

### Collect an Order

<img width="800" alt="image" src="https://github.com/Bal7hazar/yet-another-swap/assets/97087040/88136387-8968-4877-a90e-c9e6f12071c6">

To perform the collect, we just need to check that the order exist and is active (atribute belonging to Order struct).

If the Order is not active or the Order does not exist, both case will set a active status to 0 which is false.

Then we check if the OrderKey is still stored in the OrderKey array, and if it isn't so it's mean a swap remove that Order from the OrderKey array and then has been filled.

If the Order has been fill we update the position with the Ordered tick and then burn the position (we trick the Slot0 to set the Order tick).

Otherwise, we just burn the position normally (which is a closure action).

Finally we remove the Order from the Order map.

## 💚 Tests

I implemented tests including swaps that trigger or not the orders.

I also added many expected failing tests to ensure edge cases are managed correctly, but I could ;iss some.

## 🚧 Limitations

The strategy to flag all Order in a single storage value is very efficient but I doesn't provide a way to fill order partially or even consider to fill them in a FCFS way. Basically if the next tick is not reached none order is filled even if their liquidity is used, but I think this is an acceptable edge case regarding all the gas saved regarding the subgroup strategy.

I strictly implement the 2 asked functions, but it would be interesting to add a view to see if an Order is filled or not.

Also the collect/cancel function should be split to catch the user expectation correctly I think.
