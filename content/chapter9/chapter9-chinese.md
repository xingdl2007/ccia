#第9章 高级线程管理

**本章主要内容**

- 线程池<br>
- 处理线程池中任务的依赖关系<br>
- 池中线程如何获取任务<br>
- 中断线程<br>

在之前的章节中，我们通过创建`std::thread`对象来对每一个线程进行管理。在一些地方，你已经看到这种方式是不可取的了，因为需要在线程的整个生命周期中对其进行管理，根据当前使用的硬件来决定恰当的线程数量，等等。理想情况是将代码划分为最小块，以便能并发执行，然后将他们传递给处理器和标准库，进行性能的并行优化。

另一种情况是，当你使用多线程来解决某个问题时，当某个条件达成的时候，就可以提前结束。这可能是因为结果已经确定好了，或者因为发生了错误，亦或者是用户显式的终止操作。无论是哪种原因，线程都需要发送“请停止”的请求，放弃给定的任务，清理，然后尽快停止运行。

在本章，我们将了解一下管理线程和任务的机制，从自动管理线程数量和自自动管理划分任务开始。

##9.1 线程池

在很多公司里面，雇员通常会在办公室度过他们的办公时光(偶尔也会外出访问客户或供应商)，或是参加贸易展会。虽然这些路程可能很有必要，并且可能需要很多人一起去，不过对于一些比较特别的雇员来说，这一去就是几个月，甚至是几年。公司要给每个雇员都配一辆车，这基本上是不可能的，不过公司可以提供一些共用车辆；这样就会有一定数量车，来让所有雇员使用。当一个员工要去异地旅游时，那么他就可以从共用车辆中预定一辆，并在返回公司的时候将车交还。如果有一天共用车辆没有闲置的，雇员就不得不将其旅程后延了。

线程池就是类似的一种方式。在大多数系统中，将每个任务制定给一个线程是不切实际的，不过可以利用现有的并发可能，进行并发。线程池就提供了这样的功能；提交到线程池中的任务将会被并发执行，提交的任务将会挂在任务队列上。队列中的每一个任务都会被池中的一个工作线程所获取，当任务执行完成后，再回到线程池中获取下一个任务。

创建一个线程池的时候，会遇到几个关键的设计问题，比如，可使用的线程数量，高效的任务分配方式，以及是否需要等待一个任务完成。在本节，我们将看到很多线程池的实现是如何解决这些问题的，让我们从最简单的线程池开始吧！

###9.1.1 最简单的线程池

作为最简单的线程池，其拥有固定数量的工作线程(通常工作线程的数量是与`std::thread::hardware_concurrency()`相同的)。当有工作需要完成，可以调用函数将任务挂在任务队列中。每个工作线程都会从任务队列上获取任务，然后执行这个任务，执行完成后再回来获取新的任务。在最简单的情况下，线程就不需要等待其他线程完成对应任务了。如果需要等待，那么就需要对同步进行管理。

下面清单中的代码就展示了一个最简单的线程池实现。

清单9.1 简单的线程池
```c++
class thread_pool
{
  std::atomic_bool done;
  thread_safe_queue<std::function<void()> > work_queue;  // 1
  std::vector<std::thread> threads;  // 2
  join_threads joiner;  // 3

  void worker_thread()
  {
    while(!done)  // 4
    {
      std::function<void()> task;
      if(work_queue.try_pop(task))  // 5
      {
        task();  // 6
      }
      else
      {
        std::this_thread::yield();  // 7
      }
    }
  }

public:
  thread_pool():
    done(false),joiner(threads)
  {
    unsigned const thread_count=std::thread::hardware_concurrency();  // 8

    try
    {
      for(unsigned i=0;i<thread_count;++i)
      {
        threads.push_back( 
          std::thread(&thread_pool::worker_thread,this));  // 9
      }
    }
    catch(...)
    {
      done=true;  // 10
      throw;
    }
  }

  ~thread_pool()
  {
    done=true;  // 11
  }

  template<typename FunctionType>
  void submit(FunctionType f)
  {
    work_queue.push(std::function<void()>(f));  // 12
  }
};
```

这个实现中有一组工作线程②，并且使用了一个线程安全队列(从第6章)①来管理任务队列。这种情况下，用户不用等待任务，并且这些函数不需要返回任何值，所以这里可以使用`std::function<void()>`来封装任务。submit()函数会对函数或可调用对象包装成一个`std::function<void()>`实例，并将其推入队列中⑫。

线程始于构造函数：使用`std::thread::hardware_concurrency()`来获取硬件支持多少个并发线程⑧，折现线程会在worker_thread()成员函数中执行⑨。

当有异常抛出，线程启动就会失败，所以需要保证任何已启动的线程都会停止，并且能在这种情况下清理的十分干净。当有异常抛出时，通过使用*try-catch*来设置done标识⑩，还有join_threads类的实例(来自于第8章)③用来汇聚所有线程。这里当然也需要析构函数：仅设置done标识⑪，并且join_threads可以确保所有线程在线程池销毁前，全部执行完成。注意成员声明的顺序很重要：done标识和worker_queue必须在threads数组之前声明，而这个数据必须在joiner前声明。这就能确保成员能以正确的顺序销毁；比如，所有线程都停止运行时，队列就可以安全的销毁了。

worker_thread函数本身就很简单：就是执行一个循环直到done标识被设置④，从任务队列上获取任务⑤，以及同时执行这些任务⑥。如果任务队列上没有任务，函数会调用`std::this_thread::yield()`进行休息⑦，并且给予其他线程向任务队列上推送任务的机会。

对于一些简单的情况，这样线程池就足以满足要求，特别是任务完全独立没有任何返回值，或需要执行一些阻塞操作的。不过在很多情况下，这样简单的线程池是完全不够用的，还有其他情况使用这样简单的线程池可能会出现问题，比如：死锁。同样，在简单例子中，使用`std::async`能提供更好的功能，如第8章中的那些例子一样。在本章中，我们将了解一下更加复杂的线程池实现，通过添加性能满足用户需求，或减少问题的发生几率。首先，从已经提交的任务开始说起。

###9.1.2 等待提交到线程池中的任务

在第8章中的例子中，在线程间的任务划分完成后，代码会显式生成新线程，而主线程通常就是等待新生成的线程结束，确保在返回调用之前，所有任务都完成了。使用线程池，就需要等待任务提交到线程池中，而非直接提交给单个线程。这与基于`std::async`的方法(第8章等待future的例子)类似，使用清单9.1中的简单线程池，这里必须使用第4章中提到的技术：条件变量和future。这就会增加代码的复杂度；不过，这要比直接对任务进行等待的方式好的多。

通过增加线程池的复杂度，你可以直接等待任务完成。可以使用submit()函数返回一个对任务描述的句柄，并用来等待任务的完成。任务句柄会用条件变量或future进行包装，这样使用线程池来简化代码。

一种特殊的情况就是，执行任务的线程需要返回一个结果到主线程上进行处理。你已经在本书中看到多这样的例子，比如parallel_accumulate()函数(第2章)。在这种情况下，需要使用future来对最终的结果进行转移。清单9.2就展示了对简单线程池的一些修改，这样就能等待任务完成，以及在在工作线程完成后，返回一个结果到等待线程中去不过`std::packaged_task<>`实例是不可拷贝的，仅是可移动的，所以这里不能再使用`std::function<>`来实现任务队列，因为`std::function<>`需要存储可复制构造的函数对象。或者，包装一个自定义函数，用来处理只可移动的类型。这就是一个带有函数操作符的类型擦除类。只需要处理那些没有函数和无返回的函数，所以这是一个简单的虚函数调用。

清单9.2 可等待任务的线程池
```c++
class function_wrapper
{
  struct impl_base {
    virtual void call()=0;
    virtual ~impl_base() {}
  };

  std::unique_ptr<impl_base> impl;
  template<typename F>
  struct impl_type: impl_base
  {
    F f;
    impl_type(F&& f_): f(std::move(f_)) {}
    void call() { f(); }
  };
public:
  template<typename F>
  function_wrapper(F&& f):
    impl(new impl_type<F>(std::move(f)))
  {}

  void operator()() { impl->call(); }

  function_wrapper() = default;

  function_wrapper(function_wrapper&& other):
    impl(std::move(other.impl))
  {}
 
  function_wrapper& operator=(function_wrapper&& other)
  {
    impl=std::move(other.impl);
    return *this;
  }

  function_wrapper(const function_wrapper&)=delete;
  function_wrapper(function_wrapper&)=delete;
  function_wrapper& operator=(const function_wrapper&)=delete;
};

class thread_pool
{
  thread_safe_queue<function_wrapper> work_queue;  // 使用function_wrapper，而非使用std::function

  void worker_thread()
  {
    while(!done)
    {
      function_wrapper task;
      if(work_queue.try_pop(task))
      {
        task();
      }
      else
      {
        std::this_thread::yield();
      }
    }
  }
public:
  template<typename FunctionType>
  std::future<typename std::result_of<FunctionType()>::type>  // 1
    submit(FunctionType f)
  {
    typedef typename std::result_of<FunctionType()>::type
      result_type;  // 2
    
    std::packaged_task<result_type()> task(std::move(f));  // 3
    std::future<result_type> res(task.get_future());  // 4
    work_queue.push(std::move(task));  // 5
    return res;  // 6
  }
  // 休息一下
};
```

首先，修改的是submit()函数①返回一个`std::future<>`保存任务的返回值，并且允许调用者等待任务完全结束。这就需要你知道提供函数f的返回类型，这就是使用`std::result_of<>`的原因：`std::result_of<FunctionType()>::type`是FunctionType类型的引用实例(如f)，并且没有参数。同样，在函数中可以对result_type typedef②使用`std::result_of<>`。

然后，将f包装入`std::packaged_task<result_type()>`③，因为f是一个无参数的函数或是可调用对象，能够返回result_type类型的实例。在向任务队列推送任务⑤和返回future⑥前，就可以从`std::packaged_task<>`中获取future了④。注意，要将任务推送到任务队列中时，只能使用`std::move()`，因为`std::packaged_task<>`是不拷贝的。为了对任务进行处理，队列里面存的就是function_wrapper对象，而非`std::function<void()>`对象。

现在线程池允许你等待任务，并且返回任务后的结果。下面的清单就展示了，如何让parallel_accumuate函数使用线程池。

清单9.3 parallel_accumulate使用一个可等待任务的线程池
```c++
template<typename Iterator,typename T>
T parallel_accumulate(Iterator first,Iterator last,T init)
{
  unsigned long const length=std::distance(first,last);
  
  if(!length)
    return init;

  unsigned long const block_size=25;
  unsigned long const num_blocks=(length+block_size-1)/block_size;  // 1

  std::vector<std::future<T> > futures(num_blocks-1);
  thread_pool pool;

  Iterator block_start=first;
  for(unsigned long i=0;i<(num_blocks-1);++i)
  {
    Iterator block_end=block_start;
    std::advance(block_end,block_size);
    futures[i]=pool.submit(accumulate_block<Iterator,T>());  // 2
    block_start=block_end;
  }
  T last_result=accumulate_block<Iterator,T>()(block_start,last);
  T result=init;
  for(unsigned long i=0;i<(num_blocks-1);++i)
  {
    result+=futures[i].get();
  }
  result += last_result;
  return result;
}
```

与清单8.4相比，有几个点需要注意一下。首先，工作量是依据使用的块数(num_blocks①)，而不是线程的数量。为了利用线程池的最大化可扩展性，需要将工作块划分为最小工作块，这是为了让工作并发起来。当线程池中线程不多时，每个线程将会处理多个工作块，不过随着硬件可用线程数量的增长，会有越来越多的工作块并发执行。

当你选择“因为能并发执行，最小工作块值的一试”时，就需要谨慎了。向线程池提交一个任务是有一定的开销的，让工作线程执行这个任务，并且将返回值保存在`std::future<>`中，对于太小的任务，这样的开销是不划算的。如果你的任务块太小，使用线程池的速度可能都不及单线程的速度。

假设，任务块的大小是合理的，那么就不用为这些事而担心：打包任务、获取future或存储之后要汇入的`std::thread`对象；在使用线程池的时候，这些都需要注意。之后，你说要做的就是调用submit()来提交你的任务②。

线程池也需要注意异常安全。任何异常都会通过submit()返回给future，并在获取future的结果时，抛出异常。如果函数因为异常退出，线程池的析构函数丢掉那些没有完成的任务，等待线程池中的工作线程完成工作。

在简单的例子中这个线程池工作的还算不错，因为这里的任务都是相互独立的。不过，当任务队列中的任务有依赖关系时，这个线程池就不能胜任工作了。

###9.1.3 等待依赖任务

以之前的快速排序算法为例，原理很简单：数据与中轴数据项比较，在中轴项两侧分为大于和小于的两个序列，然后再对这两组序列进行排序。这两组序列会递归排序，最后会整合成一个全排序序列。要将这个算法写成并发模式，需要保证递归调用能够使用硬件的并发能力。

回到第4章，我们第一次接触这个例子，我们使用`std::async`来执行每一层的调用，让标准库来选择，是在新线程上执行这个任务，还是当对应get()调用时，同步执行。工作起来很不错，因为诶一个任务都在其自己的线程上执行，或当需要的时候进行调用。

当我们回顾第8章时，使用了一个使用固定数量线程(根据硬件可用并发线程数)的结构体。在这样的情况下，使用了栈来挂起要排序的数据块。当每个线程在为一个数据块排序前，会向数据栈上添加一组要排序的数据，然后在对当前数据块排序结束后，直接对另一块进行排序。这里，等待其他线程完成排序，可能会造成死锁，因为这会消耗有限的线程。有一种情况很可能会出现，那就是所有线程都在等某一个数据块被排序，不过没有线程在做排序。我们通过拉取栈上数据块的线程，并对数据块进行排序，来解决这个问题；因为，这个已处理的特殊数据块，就是其他线程都在等待排序数据块。

如果只是用简单的线程池进行替换，例如第4章替换`std::async`的那个线程池。现在只有限制数量的线程，因为线程池中没有空闲的线程，线程会等待没有被安排的任务。因此，需要一个和第8章中类似的解决方案：当等待某个数据块完成时，去处理未完成的数据块。如果是使用线程池来管理任务列表和相关线程——要使用线程池的主要原因——你就不用再去访问任务列表了。可以对线程池做一些改动，让其来自动完成这些事情。

最简单的方法就是在thread_pool中添加一个新函数，来执行任务队列上的任务，并对线程池进行管理。高级线程的实现可能会在等待函数中添加逻辑，或等待其他函数来处理这个任务，优先的任务会让其他的任务进行等待。下面清单中的实现，就展示了一个新的run_pending_task()函数，对于快速排序的修改将会在清单9.5中展示。

清单9.4 run_pending_task()函数实现
```c++
void thread_pool::run_pending_task()
{
  function_wrapper task;
  if(work_queue.try_pop(task))
  {
    task();
  }
  else
  {
    std::this_thread::yield();
  }
}
```

run_pending_task()的实现去掉了在worker_thread()函数中的主循环。这个函数就是在任务队列中有任务的时候，执行任务；要是没有的话，就会让操作系统对线程重新进行分配。下面对快速排序算法的实现要比清单8.1中版本简单许多，这是因为所有线程管理逻辑都被移入线程池中了。

清单9.5 基于线程池的快速排序实现
```c++
template<typename T>
struct sorter  // 1
{
  thread_pool pool;  // 2

  std::list<T> do_sort(std::list<T>& chunk_data)
  {
    if(chunk_data.empty())
    {
      return chunk_data;
    }

    std::list<T> result;
    result.splice(result.begin(),chunk_data,chunk_data.begin());
    T const& partition_val=*result.begin();
    
    typename std::list<T>::iterator divide_point=
      std::partition(chunk_data.begin(),chunk_data.end(),
                     [&](T const& val){return val<partition_val;});

    std::list<T> new_lower_chunk;
    new_lower_chunk.splice(new_lower_chunk.end(),
                           chunk_data,chunk_data.begin(),
                           divide_point);

    std::future<std::list<T> > new_lower=  // 3
      pool.submit(std::bind(&sorter::do_sort,this,
                            std::move(new_lower_chunk)));

    std::list<T> new_higher(do_sort(chunk_data));

    result.splice(result.end(),new_higher);
    while(!new_lower.wait_for(std::chrono::seconds(0)) ==
      std::future_status::timeout)
    {
      pool.run_pending_task();  // 4
    }

    result.splice(result.begin(),new_lower.get());
    return result;
  }
};

template<typename T>
std::list<T> parallel_quick_sort(std::list<T> input)
{
  if(input.empty())
  {
    return input;
  }
  sorter<T> s;

  return s.do_sort(input);
}
```

和清单8.1相比，这里将实际工作放在sorter类模板的do_sort()成员函数中执行①，即使在这个例子中仅对thread_pool实例进行包装②。

你的线程和任务管理，就会减少向线程池中提交一个任务③，并且在线程等待的时候，执行任务队列上未完成的任务④。这里的实现要比清单8.1简单许多，这里需要显式的管理线程和栈上要排序的数据块。当有任务提交到线程池中，可以使用`std::bind()`绑定this指针到do_sort()上，绑定是为了让数据块进行排序。在这种情况下，需要对new_lower_chunk使用`std::move()`来将其传入函数，数据移动要比拷贝的方式开销少。

虽然，现在使用等待其他任务的方式，解决了死锁问题，这个线程池距离理想的线程池很远。首先，每次对submit()的调用和对run_pending_task()的调用，访问的都是同一个队列。在第8章中，我们了解过，当多线程去修改一组数据，就会对性能有所影响，所以需要来解决这个问题。

###9.1.4 避免任务队列的竞争

###9.1.5 获取任务

##9.2 中断线程

###9.2.1 启动和中断其他线程

###9.2.2 检查线程是否中断

###9.2.3 中断条件变量等待

###9.2.4 中断`std::condition_variable_any`的等待

###9.2.5 中断其他阻塞调用

###9.2.6 处理中断

###9.2.7 应用退出时中断后台任务

##9.3 小结