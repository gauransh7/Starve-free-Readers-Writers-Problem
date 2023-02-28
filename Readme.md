# Starve Free Reader-Writer's problem

The Reader-Writer's problem involves synchronizing multiple processes that can be classified by 2 categories:

- **Readers -** Process to read data from a shared memory location
- **Writers -** Process to write data to the shared memory location

Since the shared memory is common to both process, we need to ensure Process Synchronization to maintain the consistency of data.

## Implementation using Semaphore

```cpp
// Define the PCB struct to hold process details
struct PCB {
    int process_id;
    string process_name;
    int priority;
    // Other details as needed
};

// Define the Node struct to hold the PCB and a pointer to the next node
struct Node {
    PCB pcb;
    Node* next;
};

// Define the Queue class to manage the list of nodes
class Queue {
    private:
        Node* head;
        Node* tail;

    public:
        // Constructor to initialize an empty Queue
        Queue() {
            head = NULL;
            tail = NULL;
        }

        // Method to add a new PCB to the end of the Queue
        void push(PCB new_pcb) {
            Node* new_node = new Node;
            new_node->pcb = new_pcb;
            new_node->next = NULL;
            if (head == NULL) {
                head = new_node;
                tail = new_node;
            } else {
                tail->next = new_node;
                tail = new_node;
            }
        }

        // Method to remove the PCB at the front of the Queue
        void pop() {
            if (head == NULL) {
                return; // Queue is already empty
            }
            Node* temp = head;
            head = head->next;
            delete temp;
            if (head == NULL) {
                tail = NULL;
            }
        }

};
```

### The Semaphore
```cpp
struct Semaphore{

    int semaphore = 1; // The actual value of the semaphore
    
    // construtor to initialize value to 1
    Semaphore(){
        semaphore = 1;
    }

    Semaphore(int val){
        semaphore = val;
    }

    // A queue to store the waiting processes
    Queue *FIFO = new Queue();

    // A function to implement the 'wait' functionality of a semaphore
    void wait(int pID){
        semaphore--;

        // If all resources are busy, push the process into the waiting queue and 'block' it
        if(semaphore < 0){
            FIFO->push(pID);
            PCB *pcb = FIFO->rear;
            block(pcb);
        }
    }

    // A function to implement the 'signal' functionality of a semaphore
    void signal(){
        semaphore++;

        // If there are any processes waiting for execution, pop from the waiting queue and wake the process up
        if(semaphore <= 0){
            PCB *pcb = FIFO->pop();
            wakeup(pcb);
        }
    }

};
```

### Writers Starve solution
In this solution to the reader-writer's problem, there is a chance that the writers process will starve. 

The global variables (that are shared across all the readers and writers) are as shown below.

#### Global Variables
```cpp
Semaphore *mutex = new Semaphore(1);
Semaphore *rw_mutex = new Semaphore(1);
int counter = 0;
```
There are 2 mutex locks implemented using Semaphores namely `mutex` and `rw_mutex`. `mutex` ensures the mutual exclusion of readers while accessing the variable `counter` and `rw_mutex` ensures that all the writers get access to the shared memory resource exclusively. The implementation of the reader is shown below

#### Implementation: Reader
```cpp
do{
    // Entry Section
    mutex->wait(processID);
    counter++;
    if(counter == 1) rw_mutex->wait(processID);
    mutex->signal();

    /**
     * 
     * Critical Section
     * 
    */

   // Exit Section
   mutex->wait(processID);
   counter--;
   if(counter == 0) rw_mutex->signal();
   mutex->signal();

    // Remainder Section

} while (true);
```
Here, if a reader is waiting for a writer process to signal the `rw_mutex`, all the other readers are waiting on `mutex`. After the writer process signals the `mutex`, all the readers can simultaneously perform the read operations. Till all the readers are done, all the writers are paused on `rw_mutex`, thus, causing starvation.

#### Implementation: Writer
```cpp
do{
    // Entry Section
    rw_mutex->wait(processID);
    
    /**
     * 
     * Critical Section
     * 
    */

   // Exit Section
   rw_mutex->signal();

   // Remainder Section

} while (true);
```


### Commonly Used Solution (Starve Free)

In the aforementioned method, writer process starve because there was nothing that stops readers from repeatedly accessing the the common resource. In this approach, a new mutex lock is shown that is implemented utilising the semaphore `in_mutex`. The procedure shown in the above solution can be entered by the process that has access to this mutex lock, giving it access to the resource. As all processes are pushed into the FIFO queue of the semaphore `in_mutex`, this implements a check to the readers who arrive after the writers. As a result, this algorithm does not starve.

#### Global Variables
```cpp
Semaphore *mutex = new Semaphore(1);
Semaphore *rw_mutex = new Semaphore(1);
Semaphore *in_mutex = new Semaphore(1);

int counter = 0;
```
The rest of the variables are same as the first solution with an addition of `in_mutex`. Now, both the reader and the writer implementations have to enclosed within `in_mutex` to ensure mutual exclusion in the whole process and thus, making the algorithm starve-free.

#### Implementation: Reader
```cpp
do{
    
    // Entry Section
    in_mutex->wait(processID);
    mutex->wait(processID);
    counter++;
    if(counter == 1) rw_mutex->wait(processID);
    mutex->signal();
    in_mutex->signal();

    /**
     * 
     * Critical Section
     * 
    */

   // Exit Section
   mutex->wait(processID);
   counter--;
   if(counter == 0) rw_mutex->signal();
   mutex->signal();

   // Remainder Section

} while (true);
```
Initially, `wait()` function is called for `in_mutex`. If a reader is waiting for a writer process, the reader is queued in the FIFO queue of the `in_mutex` (rather than `mutex`) with the fellow writers. Thus, in_mutex acts as a medium which ensures that all the processes have the same priority irrespective of their type being reader or writer.

#### Implementation: Writer
```cpp
do{

    // Entry Section
    in_mutex->wait(processID);
    rw_mutex->wait(processID);

    /**
     * 
     * Critical Section
     * 
    */

    // Exit Section
    rw_mutex->signal();
    in_mutex->signal();

    // Remainder Section

} while (true);
```
Like the readers, the writers are also queued in the FIFO queue of `in_mutex` by calling `wait()` for `in_mutex`.

Hence, all the processes requiring access to the resources can be scheduled in a FCFS manner.