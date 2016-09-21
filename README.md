# Memory-Dependence-Predictor on Gem 5
===
Team Member: Yuting Liu, and Danni Wang

## Specification

Since Gem5 platform file is really too large to fit in github, I only extract the memory repository where we implement different technologies of memory dependence predictor.

To know more about the Gem5, please check [HERE](http://gem5.org/Main_Page)

After downloading GEM5 system, please rename `mem_load_pair/` and `mem_store_set/` into `mem/`, to substitute `src/mem/`.

## Performance
We use with [SPEC CPU 2006](https://www.spec.org/cpu2006/) to judge the performance of our implementations.

## Implementations

### Naive
Gem5 has simple implementation to scratch the memory reference directly. We use this implementation works as our basic implementation of performance measurement.

### Non-Speculation
This predictor allows the load instructions to execute out-of-order commiting memory-order violations, so execute loads when their register dependencies were ready, independent of their memory dependencies.

The advantage is no false dependence. However, it incurs great performance penalty on memory-order violations, called memory trap penalty, a term as the number of cycle between the first time a load is fetched to the next time the load fetched after a memory-order violation is detected. Although the memory trap penalty can be reduced by re-executing only the load instruction and its dependent instructions, memory-order violations still negatively impact performance by occupying the entries in instruction queue and thus reducing the instruction level parallelism.

### Store Load Pair
The store-load pair speculation is to dynamically indentify the store-load pairs that are likely to be data dependent and then synchronize the instruction pairs. For the first part, past history is used to identify and track the store-load pairs. For synchronization, a condition variable is used to build the association between the store-load pair, which indicates that the load with true dependences can only be executed after the stores signal it.
  
Based the mechanism discussed above, the store-load pair predictor can be implemented with two tables, the hardware structures that cache the information for history and synchronization. Memory dependence prediction table (MDPT) whose fields consist of valid bit, load PC and store PC, contains the history of mis-speculations. It is used to identify the store-load pairs, each entry predicting that the load instruction is likely to depend on the store instructions. Considering the limited size of MDPT and the number of mis-speculation pairs, some replacement policy should be applied and thus random replacement policy is chosen in our study.

Memory dependence synchronization table (MDST) is the other table for the implementation of store-load pair predictor. It provides a condition variable to synchronized the instruction pair that is detected by the MDPT. Specifically, the fields in MDST include valid bit, load PC, store PC, load ID, store ID, tag and empty/full flag. The load ID and store ID are the instruction sequence numbers, used for the return value of the predictor that specifies which instruction should wait and which instruction should issue. The tag is the data address that the memory instruction access, which is important for the mapping between MDPT and MDST. The tag, together with the load PC and store PC, specifies the dependence edge. This dependence edge can distinguish between the different dynamic instances and thus uniquely map the consition variable to dynamic dependences. Finally, the empty/full field provides the function of condition variable, whose value determines whether a load instruction can issue or wait for its dependent stores. Figure2 shows the hardware structures required to support the store-load pair predictor.

### Store Sets
Store sets are mainly based upon two assumptions:

(1) The historic behavior of memory-order violation can be well learned to predict the future memory dependencies.

(2) The memory dependence is limited in the scope of one store corresponding to multiple stores and multiple dependent loads on the same store. 

Store Sets are indexed by instruction PC. At the start, all the entries of Store Set Identification Table (SSIT) is invalid. If the memory-order violation is committed, the belonging store set would be created in the SSIT with allocated Store Set ID (SSID). SSIT would add the comming store and load instructions to its corresponding store set by SSID, which is used to access and update the entry in Last Fetched Store Table (LFST). LFST is a table that maintains the dynamic information about the most recently fetched store for each store set. Store inum in LFST uniquely identifies the instance of each instruction in flight.

If the coming instruction is the load, it would get a valid store set with the corresponding valid SSID. Then it will access the LFST and fetch the inum of the most recently fetched store instruction to decide whether the instruction can be issued with the resolved memory dependence. 
Considering another situation of the coming store, its found valid SSID will lead the store instruction to its valid store set. The dependence would be forced between the new store and the store instructions found in LFST. Obviously, the information about the last recently fetched store should be updated with the new table and the coming store will insert its own inum into the LFST.

The specific mechanism is required to solve the conflicts when the relationship between dependent load and store instructions are not one-to-one. When memory-order violation occurs, if neither SSID of the load and store instructions is valid, the new store set will be created and then assigned to both. In another situation where the load’s SSID is valid and the store’s SSID is not, the store will inherit the SSID from the load and become part of the load’s store set. Otherwise, overwritting the store’s old SSID with the new one is proved to be efficient to implemnt the store set. However, this rule inheriently limits the number of store sets to which one store instruction could belong, which may create new possibilities for memory-order violations. 

## Report
To know more about our project, please see the [report]() here with more specific implementation details and our performance analysis.
