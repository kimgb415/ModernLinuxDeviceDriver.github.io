---
layout: post
title:  "Chapter 6.2 Advanced Char Drivers - "
date: 2024-01-20 10:46:00 +0900   
categories: Chapter 6
---

> 💡Full code in ...

## scull pipe device

Most of the code written for scull pipe device is still compatible with modern linux kernel, so this post will focus on the details and explanations of the code.

### Device Initialization

Device initialization function handles the character device registration and initialization of some of data structures like `wait_queue_head_t` and `semaphore`. Note that the buffers of devices are lazily allocated upon the `scull_p_open` but not on initialization. 

{%- highlight c -%}
int scull_p_init(dev_t first_dev)
{
    int result = register_chrdev_region(first_dev, scull_p_nr_deivces, "scullp");
    if (result < 0) {
        pr_warn("Unable to get scullp region, error %d\n", result);
        return 0;
    }
    
    scull_p_devno = first_dev;
    
    // Note that scull_p_devices is array of struct instead of array of struct pointer
    scull_p_devices = kzalloc((sizeof(struct scull_pipe) * scull_p_nr_deivces),GFP_KERNEL);
    if (scull_p_devices == NULL) {
        unregister_chrdev_region(first_dev, scull_p_nr_deivces);
        return 0;    
    }

    for (int i = 0; i < scull_p_nr_deivces; ++i) {
        struct scull_pipe *curr_dev = &scull_p_devices[i];
        sema_init(&curr_dev->sem, 1);
        init_waitqueue_head(&curr_dev->write_queue);
        init_waitqueue_head(&curr_dev->read_queue);
        curr_dev->buffer_size = scull_p_buffer;
        scull_p_setup_cdev(curr_dev, i);
    }

    return scull_p_nr_deivces;
}
{%- endhighlight -%}


> 🥲 Though shame on me, the subtle but critical mistake I made was the wrong memory allocation of `scull_p_devices` with `sizeof(struct scull_pipe *) * scull_p_nr_deivces` instead of `sizeof(struct scull_pipe) * scull_p_nr_deivces`. This buggy code led to the less memory allocated than needed for the array of structures, hence corrupting the kernel memory space and leading to the undefined behaviour of kernel.


### Device Cleanup

Cleaning up the scull pipe device is all about freeing the allocated memory areas inside out as below. Don't forget to mark the freed pointer as `NULL` to prevent the accidental memory access.

{%- highlight c -%}
void scull_p_cleanup(void)
{
    if (!scull_p_devices)
        return;

    for (int i = 0; i < scull_p_nr_deivces; ++i) {
        struct scull_pipe *curr_dev = &scull_p_devices[i];
        cdev_del(&curr_dev->cdev);
        if (curr_dev->buffer_start) {
            kfree(curr_dev->buffer_start);
            curr_dev->buffer_start = NULL;
        }
    }

    unregister_chrdev_region(scull_p_devno, scull_p_nr_deivces);
    kfree(scull_p_devices);
    scull_p_devices = NULL;

    return;
}
{%- endhighlight -%}

### Open Device

Upon openning the scull device, make sure we store the pointer of `scull_pipe` data structure into `filp->private_data` for later manipulation of device upon read of write.

{%- highlight c -%}
struct scull_pipe *dev;

dev = container_of(inode->i_cdev, struct scull_pipe, cdev);
filp->private_data = dev;
{%- endhighlight -%}


We have to record if the device is opened for write or read so that we will only destroy the buffer upon release if no readers nor writers exist.

{%- highlight c -%}
if (filp->f_flags & FMODE_READ)
    dev->readers_cnt++;
if (filp->f_flags & FMODE_WRITE)
    dev->writers_cnt++;
{%- endhighlight -%}

With above details in mind, opening the scull pipe device is straightforward.

{%- highlight c -%}
static int scull_p_open(struct inode *inode, struct file *filp)
{
	struct scull_pipe *dev;
	
	dev = container_of(inode->i_cdev, struct scull_pipe, cdev);
	filp->private_data = dev;

    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

    if (!dev->buffer_start) {
        dev->buffer_start = kzalloc(dev->buffer_size, GFP_KERNEL);
        if (!dev->buffer_start)
            return -ENOMEM;
        dev->read_pointer = dev->write_pointer = dev->buffer_start;
        dev->buffer_end = dev->buffer_start + dev->buffer_size;
    }

    if (filp->f_flags & FMODE_READ)
        dev->readers_cnt++;
    if (filp->f_flags & FMODE_WRITE)
        dev->writers_cnt++;

    up(&dev->sem);
    return 0;
}
{%- endhighlight -%}

### Release Device

As mentioned above, scull pipe driver will simply discard all the remaining contents in the buffer and free the memory if no one is using the device any more.

{%- highlight c -%}

static int scull_p_release(struct inode *inode, struct file *filp)
{
    struct scull_pipe *dev = filp->private_data;

    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

    if (filp->f_flags & FMODE_READ)
        dev->readers_cnt--;
    if (filp->f_flags & FMODE_WRITE)
        dev->writers_cnt--;

    // discard remaining contents if no more consumers nor producers exist
    if (dev->writers_cnt == 0 && dev->readers_cnt == 0) {
        if (dev->buffer_start)
            kfree(dev->buffer_start);
        dev->buffer_start = NULL;
    }

    up(&dev->sem);
    return 0;
}
{%- endhighlight -%}

> 💡 I do consider this implementation of device release to be valid. However, there is a 'wired' scenario in which, should a writer successfully write content while no reader is waiting, the content will always be lost.


### Scull Pipe Read & Write

The essence of Blocking I/O is that user processes performing I/O, which will be available in the future, will be blocked and then awakened. Readers waiting for content and writers waiting for free space share similarities in that:
- they release the semaphore and put themselves to sleep if the condition is not met,
- upon being awakened by the other party, they acquire the semaphore and double-check if the condition remains valid.

Blocking IO for the reader is shown as an example here.

{%- highlight c -%}
if (down_interruptible(&dev->sem))
    return -ERESTARTSYS;

while (dev->write_pointer == dev->read_pointer) {
    up(&dev->sem);
    if (filp->f_flags & O_NONBLOCK)
        return -EAGAIN;

    PDEBUG("pipe reader %s is about to sleep\n", current->comm);
    // signal the wait event is interrupted
    if (wait_event_interruptible(dev->read_queue, dev->write_pointer != dev->read_pointer))
        return -ERESTARTSYS; 
    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;
}
{%- endhighlight -%}

> 💡 Another lesson I learned from the above implementation is that `wait_event_interruptible` is only `interruptible` if the return value of it is properly handled, which, in this case, by returning `-ERESTARTSYS`. If you're interested, try removing the if statement that wraps `wait_event_interruptible`. You'll find that the reader ends up in an endless loop of sleeping and awakening upon signals like `Ctrl+C`.

Besides blocking I/O, the tricky part of managing a scull pipe device involves handling the circular buffer. The following two points deserve particular attention:
- Writers can only write up to `buffer_size - 1` bytes, thereby leaving one byte empty to indicate the buffer is full (`write_pointer` is one byte behind the `read_pointer`).
- if either the reader or the writer goes beyond the end of the buffer, scull pipe driver will truncate the request to the end of circular buffer and reset the corresponding pointer to the beginning for the next request.

Again is the example from `file_opertaions.read` of circular buffer handling.

{%- highlight c -%}
if (dev->write_pointer >= dev->read_pointer) {
    // readers can consume all the content up to where write_pointer is
    count = min(count, (size_t) (dev->write_pointer - dev->read_pointer));
} else {
    count = min(count,  (size_t) (dev->buffer_end - dev->read_pointer));
}
if (copy_to_user(buf, (size_t) dev->read_pointer, count)) {
    // release the semaphore in case of EFAULT
    up(&dev->sem);
    return -EFAULT;
}
dev->read_pointer += count;

// wrap read pointer if reached the end of buffer
if (dev->read_pointer == dev->buffer_end)
    dev->read_pointer = dev->buffer_start;
{%- endhighlight -%}


### Minor code modifications to use scull pipe
Since, the code for scull pipe is written in `pipe.c`, we have to make sure it's compiled in Makefile.

![scull pipe makefile]({{ "/assets/img/chapter6/scull_pipe_makefile.png" | prepend: site.baseurl }})

Last but not least, don't forget to add bash scripts to make device nodes for scull pipe upon module insertion and corresponding cleanup upon module removal.

![scull pipe load]({{ "/assets/img/chapter6/scull_load_modification.png" | prepend: site.baseurl }})

### Testing non-blokcing IO

I wrote a simple C program called `non-blocking-test.c` by mimicking `nbtest.c` to test the non-blocking IO behavior of the scull pipe device. It simply opens the pipe device and marks the file descriptor with `O_NONBLOCK`, then attempts to read its content. If no contents are available, it will sleep for a second before the next read attempt.

{%- highlight c -%}
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>
#define BUFFER_SIZE 4096

char buffer[BUFFER_SIZE];

int main(int argc, char **argv)
{
    int fd = open("/dev/scullpipe0", O_RDONLY);
    if (fd == -1) {
        perror("Failed to open scull pipe device\n");
        exit(1);
    }

    // mark file descriptor as non-blocking
    fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK);
    while (1) {
        int n = read(fd, buffer, BUFFER_SIZE);
        if (n >= 0)
            printf("[scull pipe read] %s\n", buffer);

        // break if error other than EAGAIN occurred
        if (n < 0  && errno != EAGAIN) {
            perror("Error occurred reading scull pipe device\n");
            close(fd);
            exit(1);
        } else {
            printf("Retry scull pipe read again\n");
        }

        // sleep for 1 second
        sleep(1);
    }
}
{%- endhighlight -%}

### poll syscall of scull pipe

When `file_operations.poll` is called for scull pipe device, it will register the current process into both read queue and write queue through `poll_wait` first and then calculate the poll mask. If the mask does not satisfy the one given from user space, the current user process will sleep. Upon the current process is awaken, it will recalculate the poll mask and return the result back to the user space program.

{%- highlight c -%}
// poll_table *wait can be NULL if user space program called poll with timeout value of 0
static unsigned int scull_p_poll(struct file *filp, poll_table *wait)
{
    struct scull_pipe *dev = filp->private_data;
    unsigned int mask = 0;

    down(&dev->sem);

    // add current process to both write queue and read queue
    // Note that the current process is not rescheduled immediately
    poll_wait(filp, &dev->write_queue, wait);
    poll_wait(filp, &dev->read_queue, wait);

    PDEBUG("poll scull pipe device\n");
    if (dev->read_pointer != dev->write_pointer)
        mask |= POLLIN | POLLRDNORM;
    if (space_free(dev))
        mask |= POLLOUT | POLLWRNORM;

    up(&dev->sem);
    return mask;
}
{%- endhighlight -%}

Let's test if the polling works fine with user space C program `poll_scull_pipe.c`. This user space program polls for the `POLLIN` event for 5 seconds and then informs the user about polling result.

{%- highlight c -%}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <poll.h>
#include <errno.h>
#define TIMEOUT_MS 5000


int main(int argc, char **argv)
{
    int ret_val = 1;
    struct pollfd *poll_fd = malloc(sizeof(struct pollfd));
    if (poll_fd == NULL) {
        perror("Failed to allocate memory");
        goto exit;
    }

    // make sure it's opened as non-blokcing mode
    poll_fd->fd = open("/dev/scullpipe0", O_RDONLY | O_NONBLOCK);
    if (poll_fd->fd == -1) {
        perror("Failed to open scull pipe device");
        goto exit;
    }
    poll_fd->events = POLLIN;

    int poll_result = poll(poll_fd, 1, TIMEOUT_MS);
    if (poll_result == -1) {
        perror("Failed to poll scull pipe");
    } else if (poll_result == 0) {
        perror("Timeout polling scull pipe");
    } else {
        if (poll_fd->revents & POLLIN) {
            char buffer[1024];
            ssize_t cnt = read(poll_fd->fd, buffer, sizeof(buffer) - 1);

            if (cnt == -1) {
                perror("Failed to read scull pipe");
                goto exit;
            }

            buffer[cnt] = '\0';
            printf("[scull pipe read] %s\n", buffer);
            ret_val = 0;
        }
    }

exit:
    if (poll_fd) {
        if (poll_fd->fd != -1)
            close(poll_fd->fd);
        free(poll_fd);
    }

    return ret_val;
}
{%- endhighlight -%}