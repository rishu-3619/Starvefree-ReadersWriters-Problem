# Starvefree-ReadersWriters-Problem

**The Reader-Writer's Problem Statement:**

The Reader-Writer Problem is a classic Computer Science challenge that occurs when multiple processes access the same data structure, such as a database or storage area, at the same time. The critical section of the structure can only be accessed by one writer at a time, but multiple readers can access it simultaneously. To ensure process synchronization and avoid conflicts between readers and writers, semaphores are used. However, this can lead to one group, either readers or writers, being prioritized over the other, resulting in starvation.

The problem deals with multiple processes characterised into two types:
1. Readers: They just access the resource and do not make a change to them.
2. Writers: They access the resource to make a change in them, which has to be done alone at a time, to avoid inconsistency.

Before starting with the solution, we should know what a semaphore is like. Semaphore is used to resolve process synchronization issues. A semaphore is associated with a critical section and has a queue (a FIFO structure) that maintains a list of blocked processes waiting to obtain the semaphore. When a process enters the blocked queue, it is prevented from executing. Upon receiving a signal from another process, the semaphore activates the process at the front of the blocked queue, allowing it to proceed.

**Analogous of the Process Control Block**
```struct process{ 
    process* next;
    int ID;
    bool state = true;                                     //  true represents an active state of the process while false represents inactive(blocked)
};```

**The FIFO Structure: Queue**
class waitingQueue{
   private:
    int size=0;
    int MAX_SIZE=100;                                     // max capacity of the waiting queue
    process *front, *rear;

   public:
        void push(int id) {
            if(size==MAX_SIZE){
                return;
            }
            process* p;
            p->ID=id;
            p->state=false;                               //  the process is blocked before pushing into the waitingQueue
            size=size+1;

            if(size==0){
                front=p;
                rear=p;
                return;
            }        
           
            rear->next=p;
            rear=p;
        }

        process nullProcess = {NULLptr, -1,  false};      //  defining a null process
        
        process* pop(){
            if(size==0){
                return &nullProcess;
            }
            
            process* nextProcess=front;
            front=front->next;
            size=size-1;

            return nextProcess;
        }

};

**The struct semaphore** is linked to the queue. This queue will be storing the list of processes waiting to acquire the semaphore
struct semaphore {
    int value = 1;
    waitingQueue* wait_queue = new waitingQueue();
    
    semaphore(int n) {
        value = n;                                          // n=number of resources of that particular type available for accessing
    }

};

**Analogy of system call: wakeUp call**
void wakeUp(process* p) {
    p->state = true;
}

void wait(semaphore *s, int id) {
    s=s-1;
    s->value;

    if((s->value)<0) {
        s->blocked_queue->push(id);
    }
}

void signal(semaphore* s) {
    s=s+1;
    s->value;

    if((s->value)<=0) {
        process* nextProcess = sem->wait_queue->pop();
        wakeUp(nextProcess);                               //  wakeUp the next process as it is ready to enter the critical section
    }
}



**Starve Free Solution**
The starve-free solution to the Reader-Writer problem involves the use of an additional semaphore called entry_mutex, which must be acquired by any process, whether a reader or a writer, before accessing the rw_mutex or entering the critical section directly. This solution resolves the issue of starvation by preventing readers from continuously accessing the critical section, which could previously have deprived writers of access. If a writer arrives when readers are still present in the critical section, the writer waits in the blockedQueue for the rw_mutex, which is obtained once all readers exit the critical section. The advantage of this solution is that it maintains the efficient reader access to the critical section while ensuring that readers and writers have equal priority and none are starved for access.

Initialization:

int resources=1;                                     // number of resources the processes are competing for
int read_count=0;                                    // number of readers in the critical section
semaphore* rw_mutex=new semaphore(resources);        // a writer with this semaphore has access to the critical section
semaphore* read_mutex=new semaphore(resources);      // used to access the read_count variable and prevents the conflicts while changing the read_count variable
semaphore* entry_mutex=new semaphore(resources);     // used at the begining of both the reader and writer codes
// A reader/writer first has to acquire this semaphore to enter the critical section


Reader's Implementation (Starve-Free):
This block would be called everytime a new reader arrives.
do {
   // ***** ENTRY SECTION ***** //
    
    wait(entry_mutex, processId);
    wait(read_mutex, processId);
    read_count=read_count+1;
    if(read_count==1){            // this implies that the first reader is trying to access the resource
        wait(rw_mutex, processId); 
    }   
    signal(read_mutex);           // now readers that have the entry_mutex can access the read_count
    signal(entry_mutex);          // as the reader is ready to enter the critical section, it frees the entry_mutex semaphore

  // ***** CRITICAL SECTION ***** //             //  this can be accessed by readers directly if read_count != 1

    wait(read_mutex, processId);
    read_count=read_count-1;
    signal(read_mutex);
    if(read_count == 0){
        signal(rw_mutex);
    }                               // the last reader frees the rw_mutex

  // ***** REMAINDER SECTION ***** //
  
} while(true);



Writer's Implementation (Starve-Free):
do {
    // ***** ENTRY SECTION ***** //
    
       wait(entry_mutex, processId);              // the writer also waits for the entry mutex first
       wait(rw_mutex, processId);                 // once free, writer process will acquire the rw_mutex and enter the critical section

    // ***** CRITICAL SECTION ***** //            // the critical section can be accessed by writer only and only if it has acquired the rw_mutex
    
       signal(rw_mutex);
       signal(entry_mutex);                       // once the writer is ready to enter the critical section it frees the entry_mutex
    
    // ***** REMAINDER SECTION ***** //
    
} while(true);


The approach described above successfully eliminates the issue of starvation, ensuring that neither readers nor writers are deprived of access to the critical section. As a result, this solution provides a starve-free resolution to the Reader-Writer problem.



**A Faster Starve Free Solution:**
As observed in the previous solution, the reader process needs to access and lock two semaphores each time it enters the critical section. This process can take a significant amount of time if acquiring semaphore locks is time-consuming. Therefore, an alternate starve-free approach that is faster than the previous solution can be employed.

Initialization:

int resources=1;
int in_count=0;
int out_count=0;                                        // read_count = in_count - out_count 
bool writer_waiting = false;                            // tells whether a writer is in waiting state to enter the critical section
semaphore* rw_mutex = new semaphore(resources-1);       // this would be required only if reader processes are in critical section and once the reader processes are done executing and writer process is waiting, then it is signalled to be free
semaphore* entry_mutex = new semaphore(resources);      // controls access to in_count variable
semaphore* out_mutex = new semaphore(resources);        // controls access to out_count variable


Reader's Implementation:
do {
  
  // ***** ENTRY SECTION ***** //
  
    wait(entry_mutex, processId);   
    in_count=in_count+1;
    signal(entry_mutex);   

  // ***** CRITICAL SECTION ***** //
    
    wait(out_mutex, processId);
    out_count=out_count+1;
    if((writer_waiting==true) && (in_count==out_count)){
        signal(rw_mutex);
    } 
    signal(out_mutex);
    
   // ***** REMAINDER SECTION ***** //
} while(true);



Writer's Implementation:
do {
     // ***** ENTRY SECTION ***** //
     
         wait(entry_mutex, processId);
         wait(out_mutex, processId);
         if(in_count==out_count){ 
            signal(out_mutex); 
         } //read_count==0 i.e. no readers
         else{
            writer_waiting = true;         // the writer process would have to wait
            signal(out_mutex);             // this was only to ensure sync of out_count 
            wait(rw_mutex, processId);
            writer_waiting = false;        // after acquiring rw_mutex, it is now ready to enter the critical section
         }
          
     // ***** CRITICAL SECTION ***** //
         
         signal(entry_mutex);
         
     // ***** REMAINDER SECTION ***** //
   
} while(true);


Explanation for Reader's code:

The previous solution required two mutex locks to be used for the entry section, resulting in potentially significant temporal overhead due to the blocking of processes. However, a faster starve-free approach exists that only requires the use of a single lock. In this approach, all readers and writers initially queue in the in_mutex to ensure equal priority. When a reader acquires the in_mutex, it signals its presence by incrementing the variable readers_started and immediately releasing the in_mutex. 
The only delay that a reader can experience is while waiting for the in_mutex. Readers can read concurrently, as writers are the only processes with a critical section between the wait() and signal() methods of in_mutex. Once a reader completes its critical section, it must signal the end of its resource usage by acquiring the out_mutex, incrementing the variable readers_completed, and checking if any waiting writers are present. If no readers are executing in their critical sections, it signals the writer to begin execution by calling signal() on the semaphore write_sem before releasing the out_mutex and proceeding to its remainder section. 
This approach saves time by requiring only one lock and minimizing the blocking of processes.


Explanation for Writer's Code:

In this approach, writers and readers both wait on the in_mutex initially. After acquiring it, writers then proceed to wait on the out_mutex. Once they acquire it, they compare the variables readers_started and readers_completed. If they are equal, then no reader is currently executing their critical section, and the writer can proceed with its critical section after signalling the out_mutex. Any other reader or writer is prevented from accessing the critical section by the fact that in_mutex has not been signalled yet.
If readers_started and readers_completed are not equal, the writer sets the variable writer_waiting to true, signals the out_mutex, and waits in writer_sem until all readers complete their execution. Once it acquires writer_sem, it sets writer_waiting to false and proceeds to its critical section. 
After completion of the critical section, it signals the in_mutex to announce that it has completed resource usage. The next process in the in_mutex queue can proceed.


**Correctness of Code:**
For a solution to be starve-free, it must satisfy the three criteria-- Mutual Exclusion, Progress and Bounded Waiting.

1. Mutual Exclusion:
The solution guarantees mutual exclusion in the critical section by allowing only one writer or multiple readers but not both at any given time. This is achieved using the entry_mutex semaphore, which treats reader and writer processes equally and assigns entry based on a first-come, first-served (FCFS) basis. Writers are given entry only when the critical section is free, while readers can also enter if other readers are present. This approach ensures mutual exclusion in the critical section.

2. Bounded Waiting:
Both algorithms require processes to go through a mutex before entering the critical section. This mutex maintains a First-In-First-Out (FIFO) queue of waiting processes. As a result, the waiting time for any process in the queue is finite or bounded, given a finite number of processes.

3. Progress:
Both algorithms have been designed in a way that the system cannot enter a deadlock state. Additionally, as long as the execution time is finite for all processes in their respective critical sections, progress is guaranteed as processes will continue executing after waiting in the in_mutex queue. Moreover, the code structure ensures that readers and writers take finite time with semaphores being released, allowing others to enter the critical section. As a result, the progress criterion is satisfied.


**REFERENCES:**
Operating System Concepts, Ninth Edition, Silberschatz, Galvin, Gagne
[Faster Fair Solution for the Reader-Writer Problem](https://arxiv.org/abs/1309.4507)
