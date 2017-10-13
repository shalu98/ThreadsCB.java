# ThreadsCB.java
package osp.Threads;
import java.util.Vector;
import java.util.Enumeration;
import osp.Utilities.*;
import osp.IFLModules.*;
import osp.Tasks.*;
import osp.EventEngine.*;
import osp.Hardware.*;
import osp.Devices.*;
import osp.Memory.*;
import osp.Resources.*;

/**
   This class is responsible for actions related to threads, including
   creating, killing, dispatching, resuming, and suspending threads.

   @OSPProject Threads
*/
public class ThreadCB extends IflThreadCB 
{
	    static GenericList readyThread;
		public static int lastCpuBurst; // t(n-1)
		public static int estimatedBurstTime; //t
		public long lastDispatch; //to find lastCPUBurst
	
	
	/**
       The thread constructor. Must call 

       	   super();

       as its first statement.

       @OSPProject Threads
    */
    public ThreadCB()
    {
        super();
        //initializing these values is that there are no previous values of tn-1  
        //and n-1 that you can use to compute estimatedBurstTime. 
        lastCpuBurst = 0;
        estimatedBurstTime = 0;
       lastDispatch = 0;

    }

    /**
       This method will be called once at the beginning of the
       simulation. The student can set up static variables here.
       
       @OSPProject Threads
    */
    public static void init()
    {
    	readyThread = new GenericList(); //Initialize the ready queue
    	lastCpuBurst =10;
    	estimatedBurstTime = 10;
    	
    }

    /** 
        Sets up a new thread and adds it to the given task. 
        The method must set the ready status 
        and attempt to add thread to task. If the latter fails 
        because there are already too many threads in this task, 
        so does this method, otherwise, the thread is appended 
        to the ready queue and dispatch() is called.

	The priority of the thread can be set using the getPriority/setPriority
	methods. However, OSP itself doesn't care what the actual value of
	the priority is. These methods are just provided in case priority
	scheduling is required.

	@return thread or null

        @OSPProject Threads
    */
    static public ThreadCB do_create(TaskCB task)
    {
    	//check to see if task is null & if task has max number of threds
        if(task == null || task.getThreadCount() >= MaxThreadsPerTask)
        {
        	dispatch();
        	return null;
        }

    

    
    ThreadCB newThread = new ThreadCB();
    newThread.setPriority(task.getPriority());
    newThread.setStatus(ThreadReady);
    newThread.setTask(task); //Associate the task with the thread
    
    //newThread.addThread(thread);
    int addedValue= task.addThread(newThread);
    
    if(addedValue != FAILURE)
    {
    	readyThread.append(newThread); //Append a new thread to the Ready Queue.

    	dispatch();
    
    return newThread;
    }
    else
    
	dispatch();
	return null;

    }

    
    
    
    
    
    
    
    
    
    
    
    
    /** 
	Kills the specified thread. 

	The status must be set to ThreadKill, the thread must be
	removed from the task's list of threads and its pending IORBs
	must be purged from all device queues.
        
	If some thread was on the ready queue, it must removed, if the 
	thread was running, the processor becomes idle, and dispatch() 
	must be called to resume a waiting thread.
	
	@OSPProject Threads
    */
    public void do_kill()
    {
    	
    	
    	int status = this.getStatus(); //get status of thread
        TaskCB taskToEnd = this.getTask();
        
        
        //If the thread status is ThreadReady, then remove it from the Ready Queue. If you used
        //the GenericList class then you can use remove().
        if(status == ThreadReady && readyThread.contains(this))
        {
        	readyThread.remove(this);
        }
        
        //if thread status is thread Running, pre-empt, get current thread from memory
        if(status== ThreadRunning)
        {
        	
        	preempt();
        	this.estimatedBurstTime = (int)((0.75* lastCpuBurst)+ (0.25 *lastCpuBurst));
        	if(this.estimatedBurstTime < 5 )
        	{
        		this.estimatedBurstTime = 5;
        	}
        	this.lastCpuBurst = (int) (HClock.get()-this.lastDispatch);
        	
        }
        

        this.getTask().removeThread(this);
        this.setStatus(ThreadKill);
        
        int devices = Device.getTableSize();
        int i=0;
        
        //Loop through the device table to purge any IORB associated with this thread. You
        //can/should use cancelPendingIO() for this
        
        while(devices>0)
        {
        	Device.get(i).cancelPendingIO(this);
        	devices--;
        	i++;
        }
        
        //Release any resources currently in use
        ResourceCB.giveupResources(this);
        
        dispatch();
        //check if task has any threads left
        if(taskToEnd.getThreadCount()<=0)
        {
        	taskToEnd.kill();
        }
        
        

    }
    
    
    
    
    
    
    
    
    
    
    
    
    

    /** Suspends the thread that is currenly on the processor on the 
        specified event. 

        Note that the thread being suspended doesn't need to be
        running. It can also be waiting for completion of a pagefault
        and be suspended on the IORB that is bringing the page in.
	
	Thread's status must be changed to ThreadWaiting or higher,
        the processor set to idle, the thread must be in the right
        waiting queue, and dispatch() must be called to give CPU
        control to some other thread.

	@param event - event on which to suspend this thread.

        @OSPProject Threads
    */
    public void do_suspend(Event event)
    {
    	 if(this != null)
			{
         
    	
     	if(this.getStatus()>=ThreadWaiting)
     	{ 
     		this.setStatus(this.getStatus()+1);
     		
     	}
     	else if(this.getStatus() ==ThreadRunning)
     	{ 
     		this.setStatus(ThreadWaiting); 
     		
     		preempt();
        	this.estimatedBurstTime = (int)((0.75* lastCpuBurst)+ (0.25 *lastCpuBurst));
        	if(this.estimatedBurstTime < 5 )
        	{
        		this.estimatedBurstTime = 5;
        	}
        	this.lastCpuBurst = (int) (HClock.get()-this.lastDispatch);
     		
     	    this.getTask().setCurrentThread(null);
     	}
     	
     	if(readyThread.contains(this))
         	readyThread.remove(this);
         }
         event.addThread(this);//add thread to the event queue
         dispatch();   	

    }

    
    
    
    
    
    
    
    
    
    
    
    /** Resumes the thread.
        
	Only a thread with the status ThreadWaiting or higher
	can be resumed.  The status must be set to ThreadReady or
	decremented, respectively.
	A ready thread should be placed on the ready queue.
	
	@OSPProject Threads
    */
    public void do_resume()
    {
    	if(this!=null)
        {
     	   if(this.getStatus()<ThreadWaiting)
     	   {
     		   return;
     	   }
     	   else
     	   {
     		   if(this.getStatus()==ThreadWaiting){
     			   this.setStatus(this.getStatus()-1);
     		   }
     		   if(this.getStatus()==ThreadReady)
     		   {
     			   readyThread.append(this);
     		   }
     	   }
     	   dispatch();
        }
    }
    
    
    
    
    
    
    
    
    
    

    /** 
        Selects a thread from the run queue and dispatches it. 

        If there is just one theread ready to run, reschedule the thread 
        currently on the processor.

        In addition to setting the correct thread status it must
        update the PTBR.
	
	@return SUCCESS or FAILURE

        @OSPProject Threads
    */
    
    
    //thread.lastDispatch = HClock.get();
    
    public static int do_dispatch()
    {
    	ThreadCB t = null;
    	try {
    		t = MMU.getPTBR().getTask().getCurrentThread();	
    	}
    	catch(NullPointerException e){
    		//if program goes here, the thread
    		//is not currently running, so it will
    		//try again
    	}
    	
		//return failure if readyQueue is empty
    	if (readyThread.isEmpty()){
    		MMU.setPTBR(null);
    		//t.lastCPUBurst = (int) HClock.get();
    		return FAILURE;
    	}
    	//If current thread is running
    	if (t != null) {
    		int remainingBurstTime = (int) (HClock.get() - t.lastDispatch);
    		ThreadCB t2 = new ThreadCB();
    		t2.estimatedBurstTime = 500000;
    		for (int i=0;i<readyThread.length();i++) {
    			ThreadCB temp = (ThreadCB) readyThread.getAt(i);
    			if (temp.estimatedBurstTime < t2.estimatedBurstTime) {
    				t2 = temp;
    			}
    			temp.estimatedBurstTime--;
    			readyThread.setAt(i, temp);
    		}
    		if (remainingBurstTime <= t2.estimatedBurstTime || remainingBurstTime < 2) { 
    			t.lastCpuBurst = (int) HClock.get();
    			return SUCCESS; //Note: prevLine
    		}
    		else {
	    		t.getTask().setCurrentThread(null);
	    		MMU.setPTBR(null);
		    	MMU.setPTBR(t2.getTask().getPageTable());
		    	t2.getTask().setCurrentThread(t);
		    	t2.setStatus(ThreadRunning);
		    	t2.lastCpuBurst = (int) HClock.get();
		    	return SUCCESS;
    		}
    	}
    	t.lastCpuBurst = (int) HClock.get();
    	return SUCCESS;
    	/*//Take the thread at the head of the ready queue
    	//as the one to be dispatched
    	t = (ThreadCB)readyQueue.removeHead();
    	MMU.setPTBR(t.getTask().getPageTable()); //set PTBR to point to the thread's page table
    	t.getTask().setCurrentThread(t); //set the new thread as the thread for its task
    	t.setStatus(ThreadRunning); // set status to running
    	//this is where interrupt would go if 
    	//this was RR scheduling
    	t.lastCPUBurst = (int) HClock.get();
    	return SUCCESS; 
    	**/

    }

    
    
    
    
    
    
    private static ThreadCB preempt()
    {
		ThreadCB currentThread = null;
		TaskCB currentTask=null;
		try{
			
			//Call MMU.getPTBR() to get the page table of the current thread
			currentTask= MMU.getPTBR().getTask(); //o find out which task owns the thread
    		currentThread= currentTask.getCurrentThread();//what current thread is
    	
    		if(currentThread != null)
    		{
    			currentThread.setStatus(ThreadReady); 
    			
        		currentTask.setCurrentThread(null);//set page table base register to null
        	}
    		
    	}catch( NullPointerException e){
    		
    		
    	}
    	    	    	    	
    	MMU.setPTBR(null); 
    	    	
    	return currentThread; 
    }
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    /**
       Called by OSP after printing an error message. The student can
       insert code here to print various tables and data structures in
       their state just after the error happened.  The body can be
       left empty, if this feature is not used.

       @OSPProject Threads
    */
    public static void atError()
    {
        // your code goes here

    }

    /** Called by OSP after printing a warning message. The student
        can insert code here to print various tables and data
        structures in their state just after the warning happened.
        The body can be left empty, if this feature is not used.
       
        @OSPProject Threads
     */
    public static void atWarning()
    {
        // your code goes here

    }


    /*
       Feel free to add methods/fields to improve the readability of your code
    */

}

/*
      Feel free to add local classes to improve the readability of your code
*/
