# Mount namespace

- �����ļ�ϵͳ�Ĺ��ص㡾��ô��⡿
- ���chroot��mount namespace���Ӱ�ȫ�����
- `/proc/[pid]/mouts`, `/proc/[pid]/mountinfo`,`/proc/[pid]/mountstats`����������ͬmount namespace�еĽ�������Щ�ļ��п�������������ͬ��
- ͨ��clone��������namespace�е�mount point list�Ǹ�namespace�ĸ���
- ͨ��unshare���������ص��б��ǵ�����֮ǰ��mount namespace�ĸ���
- ��`pivot_root`ϵͳ���ö������ļ�ϵͳ�ĸ���
- ��Mount Namespace�е���`mount()`��`umount()`����ֻӰ�쵱ǰNamespace�ڵ��ļ�ϵͳ

## `pivot_root`

```c
int syscall(SYS_pivot_root, const char *new_root, const char *put_old);
```

�ı���ý������ڵ�mount namespace��root mount��


# User Namespace
[[User namespace]]
User namespace���밲ȫ��ص��������Լ����ԣ�
- uid/gid��һ�����̵�uid/gid��username��������Բ�ͬ��namespace����user�û�Ȩ�ޣ�����uid=0��
- User namespace����Ƕ�ף����32��
- ���̵߳Ľ��̿��Լ��뵽��һ��user namespace�У��������CAP_SYS_ADMINȨ��
- ��Ŀ¼
- keys
- capabilities��ͨ��cloneϵͳ����`CLONE_NEWUSER`flag�������ӽ��̻����µ�usernamespace�л�ȡ����Capabilities��

- ��root���̿��Դ���User Namespace
- �û�Namespace���汻ӳ���root������Namespace����rootȨ��

- �ṩ��User IDs��Group IDs�ĸ���
- ʵ��unprivileged�������������root�û���ӳ��Ϊ������root�û���
- User Namespace�����©��
- ��capability��ì�ܡ�Ҳ��©�����ԭ��

�������Ϊ����namespace�е�uid/gid���������ϵ�uid/gid������һ��ӳ���ϵ��

![[Pasted image 20210427165751.png]]

user namespace����Ƕ�ף����32��

```
unshare -r #����һ��user namespace����ǰ�û�ӳ��Ϊroot�û�
```

����ͨ��setns����namespace��Ҫ�ڸ�namespace��ӵ��CAP_SYS_ADMINȨ�ޣ�����֮����ȡȫ����cap

ͨ��clone with clone_newuser flag�����µ�user namespace�л�ȡȫ��cap������ִ��exec�󣬻ᰴ��������ʽ����Capabilities��

clone with clone_newuser��Ϊ�ӽ�������securebits λ����������flag����


# IPC Namespace

-  SystemV IPC��POSIX��Ϣ���е�namespace��
-  ֧�ֵĿ���̷�ʽ��
	-  �����ڴ�
	-  ��Ϣ����
	-  �ź���

Docker֧��ָ��IPC namespace

# Network Namespace
- �����豸�ĸ��룬һ��ָlo,etho��ͨ��`ip link`
- IPv4��IPv6Э��ջ
- IP·�ɱ�
- ����ǽ����iptables
- ����״̬��Ϣ







# PID Namespace
- �ṩ����ID�ĸ���

��Դ���������
https://zhuanlan.zhihu.com/p/335171876

## Դ�����

```mermaid
graph LR
	task_struct --> nsproxy --> pid_namespace
```

```c
struct task_struct {
	... ...
    struct fs_struct *fs;
	struct files_struct *files;
	struct nsproxy *nsproxy;
	... ...
};
```

```c
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns_for_children;
	struct net 	     *net_ns;
	struct time_namespace *time_ns;
	struct time_namespace *time_ns_for_children;
	struct cgroup_namespace *cgroup_ns;
};
```

```c
struct pid_namespace {
	struct kref kref;
	struct idr idr;
	struct rcu_head rcu;
	unsigned int pid_allocated;
	struct task_struct *child_reaper;
	struct kmem_cache *pid_cachep;
	unsigned int level;
	struct pid_namespace *parent;
#ifdef CONFIG_BSD_PROCESS_ACCT
	struct fs_pin *bacct;
#endif
	struct user_namespace *user_ns;
	struct ucounts *ucounts;
	int reboot;	/* group exit code if this pidns was rebooted */
	struct ns_common ns;
} __randomize_layout;
```

### init���̵�pid namespace

ȫ�ֱ���`init_pid_ns`

```c
/*
 * PID-map pages start out as NULL, they get allocated upon
 * first use and are never deallocated. This way a low pid_max
 * value does not cause lots of bitmaps to be allocated, but
 * the scheme scales to up to 4 million PIDs, runtime.
 */
struct pid_namespace init_pid_ns = {
	.kref = KREF_INIT(2),
	.idr = IDR_INIT(init_pid_ns.idr),
	.pid_allocated = PIDNS_ADDING,
	.level = 0,
	.child_reaper = &init_task,
	.user_ns = &init_user_ns,
	.ns.inum = PROC_PID_INIT_INO,
#ifdef CONFIG_PID_NS
	.ns.ops = &pidns_operations,
#endif
};
EXPORT_SYMBOL_GPL(init_pid_ns);

```

#### init���̵�namespace����

�����namespace����task_struct��nsproxy��ָ��init_nsproxy
```c
struct nsproxy init_nsproxy = {
	.count			= ATOMIC_INIT(1),
	.uts_ns			= &init_uts_ns,
#if defined(CONFIG_POSIX_MQUEUE) || defined(CONFIG_SYSVIPC)
	.ipc_ns			= &init_ipc_ns,
#endif
	.mnt_ns			= NULL,
	.pid_ns_for_children	= &init_pid_ns,//ָ����init���̵�pid_namespace
#ifdef CONFIG_NET
	.net_ns			= &init_net,
#endif
#ifdef CONFIG_CGROUPS
	.cgroup_ns		= &init_cgroup_ns,
#endif
#ifdef CONFIG_TIME_NS
	.time_ns		= &init_time_ns,
	.time_ns_for_children	= &init_time_ns,
#endif
};
```


### �����ӽ��̵Ĺ���

��Ҫ�����Ķ���

```mermaid
graph LR
	copy_process --> copy_namespace --> get_nsproxy
	copy_namespace --> create_pid_namespaces --> copy_pid_ns --> create_pid_namespace

```

```c
int copy_namespaces(unsigned long flags, struct task_struct *tsk)
{
	struct nsproxy *old_ns = tsk->nsproxy;
	struct user_namespace *user_ns = task_cred_xxx(tsk, user_ns);
	struct nsproxy *new_ns;
	int ret;

	if (likely(!(flags & (CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC |
			      CLONE_NEWPID | CLONE_NEWNET |
			      CLONE_NEWCGROUP | CLONE_NEWTIME)))) {
		if (likely(old_ns->time_ns_for_children == old_ns->time_ns)) {
			get_nsproxy(old_ns);
			return 0;
		}
	} else if (!ns_capable(user_ns, CAP_SYS_ADMIN))//��������Ȩ�޼��
		return -EPERM;

	/*
	 * CLONE_NEWIPC must detach from the undolist: after switching
	 * to a new ipc namespace, the semaphore arrays from the old
	 * namespace are unreachable.  In clone parlance, CLONE_SYSVSEM
	 * means share undolist with parent, so we must forbid using
	 * it along with CLONE_NEWIPC.
	 */
	if ((flags & (CLONE_NEWIPC | CLONE_SYSVSEM)) ==
		(CLONE_NEWIPC | CLONE_SYSVSEM)) 
		return -EINVAL;

	new_ns = create_new_namespaces(flags, tsk, user_ns, tsk->fs);
	if (IS_ERR(new_ns))
		return  PTR_ERR(new_ns);

	ret = timens_on_fork(new_ns, tsk);
	if (ret) {
		free_nsproxy(new_ns);
		return ret;
	}

	tsk->nsproxy = new_ns;
	return 0;
}
```
- ���û�б����̳и����̵�namespace������get_nsproxy�����ü���+1��
- ����������create_new_namespaces

```c
/*
 * Create new nsproxy and all of its the associated namespaces.
 * Return the newly created nsproxy.  Do not attach this to the task,
 * leave it to the caller to do proper locking and attach it to task.
 */
static struct nsproxy *create_new_namespaces(unsigned long flags,
	struct task_struct *tsk, struct user_namespace *user_ns,
	struct fs_struct *new_fs)
{
	struct nsproxy *new_nsp;
	int err;

	new_nsp = create_nsproxy();
	if (!new_nsp)
		return ERR_PTR(-ENOMEM);

	new_nsp->mnt_ns = copy_mnt_ns(flags, tsk->nsproxy->mnt_ns, user_ns, new_fs);
	if (IS_ERR(new_nsp->mnt_ns)) {
		err = PTR_ERR(new_nsp->mnt_ns);
		goto out_ns;
	}

	new_nsp->uts_ns = copy_utsname(flags, user_ns, tsk->nsproxy->uts_ns);
	if (IS_ERR(new_nsp->uts_ns)) {
		err = PTR_ERR(new_nsp->uts_ns);
		goto out_uts;
	}

	new_nsp->ipc_ns = copy_ipcs(flags, user_ns, tsk->nsproxy->ipc_ns);
	if (IS_ERR(new_nsp->ipc_ns)) {
		err = PTR_ERR(new_nsp->ipc_ns);
		goto out_ipc;
	}

	new_nsp->pid_ns_for_children =
		copy_pid_ns(flags, user_ns, tsk->nsproxy->pid_ns_for_children);
	if (IS_ERR(new_nsp->pid_ns_for_children)) {
		err = PTR_ERR(new_nsp->pid_ns_for_children);
		goto out_pid;
	}

	new_nsp->cgroup_ns = copy_cgroup_ns(flags, user_ns,
					    tsk->nsproxy->cgroup_ns);
	if (IS_ERR(new_nsp->cgroup_ns)) {
		err = PTR_ERR(new_nsp->cgroup_ns);
		goto out_cgroup;
	}

	new_nsp->net_ns = copy_net_ns(flags, user_ns, tsk->nsproxy->net_ns);
	if (IS_ERR(new_nsp->net_ns)) {
		err = PTR_ERR(new_nsp->net_ns);
		goto out_net;
	}

	new_nsp->time_ns_for_children = copy_time_ns(flags, user_ns,
					tsk->nsproxy->time_ns_for_children);
	if (IS_ERR(new_nsp->time_ns_for_children)) {
		err = PTR_ERR(new_nsp->time_ns_for_children);
		goto out_time;
	}
	new_nsp->time_ns = get_time_ns(tsk->nsproxy->time_ns);

	return new_nsp;

out_time:
	put_net(new_nsp->net_ns);
out_net:
	put_cgroup_ns(new_nsp->cgroup_ns);
out_cgroup:
	if (new_nsp->pid_ns_for_children)
		put_pid_ns(new_nsp->pid_ns_for_children);
out_pid:
	if (new_nsp->ipc_ns)
		put_ipc_ns(new_nsp->ipc_ns);
out_ipc:
	if (new_nsp->uts_ns)
		put_uts_ns(new_nsp->uts_ns);
out_uts:
	if (new_nsp->mnt_ns)
		put_mnt_ns(new_nsp->mnt_ns);
out_ns:
	kmem_cache_free(nsproxy_cachep, new_nsp);
	return ERR_PTR(err);
}

```
- ����`create_nsproxy`����һ��nsproxy�ṹ��
- `copy_pid_ns`ֱ�ӵ�����`create_pid_namespace`


```c
static struct pid_namespace *create_pid_namespace(struct user_namespace *user_ns,
	struct pid_namespace *parent_pid_ns)
{
	struct pid_namespace *ns;
	unsigned int level = parent_pid_ns->level + 1;
	struct ucounts *ucounts;
	int err;

	err = -EINVAL;
	if (!in_userns(parent_pid_ns->user_ns, user_ns))
		goto out;

	err = -ENOSPC;
	if (level > MAX_PID_NS_LEVEL)
		goto out;
	ucounts = inc_pid_namespaces(user_ns);
	if (!ucounts)
		goto out;

	err = -ENOMEM;
	ns = kmem_cache_zalloc(pid_ns_cachep, GFP_KERNEL);//����������ڴ�
	if (ns == NULL)
		goto out_dec;

	idr_init(&ns->idr);

	ns->pid_cachep = create_pid_cachep(level);
	if (ns->pid_cachep == NULL)
		goto out_free_idr;

	err = ns_alloc_inum(&ns->ns);
	if (err)
		goto out_free_idr;
	ns->ns.ops = &pidns_operations;

	kref_init(&ns->kref);
	ns->level = level;
	ns->parent = get_pid_ns(parent_pid_ns);
	ns->user_ns = get_user_ns(user_ns);
	ns->ucounts = ucounts;
	ns->pid_allocated = PIDNS_ADDING;

	return ns;

out_free_idr:
	idr_destroy(&ns->idr);
	kmem_cache_free(pid_ns_cachep, ns);
out_dec:
	dec_pid_namespaces(ucounts);
out:
	return ERR_PTR(err);
}
```
- �����ڴ�
- �����е�level+1
- ����user namespace�е����ü���
- ��pid_namespace�ṹ���еĸ���������


### �����ӽ���pid

```c
struct upid {
	int nr;//�ò�pid ns��pidֵ
	struct pid_namespace *ns;
};

struct pid
{
	refcount_t count;
	unsigned int level;
	spinlock_t lock;
	/* lists of tasks that use this pid */
	struct hlist_head tasks[PIDTYPE_MAX];
	struct hlist_head inodes;
	/* wait queue for pidfd notifications */
	wait_queue_head_t wait_pidfd;
	struct rcu_head rcu;
	struct upid numbers[1];
};
```
- `level`���ý���������pid_ns��level����ʾ���pid�����ڶ��ٸ�pid namespace�пɼ���
- `numbers`���洢ÿ���pid��Ϣ�ı䳤���飬���Ⱦ��������level

copy_process�У�ͨ��alloc_pidΪ�ӽ�������һ��`struct pid`
```c
//copy_process()	
if (pid != &init_struct_pid) { //pidָ���Ǻ�����Σ���ͨ���̴���ʱ���õ�__do_fork�иò���ΪNULL
		pid = alloc_pid(p->nsproxy->pid_ns_for_children);
		if (IS_ERR(pid)) {
			retval = PTR_ERR(pid);
			goto bad_fork_cleanup_thread;
		}
	}
```
- ����`alloc_pid`

```c
struct pid *alloc_pid(struct pid_namespace *ns, pid_t *set_tid,
		      size_t set_tid_size)
{
	struct pid *pid;
	enum pid_type type;
	int i, nr;
	struct pid_namespace *tmp;
	struct upid *upid;
	int retval = -ENOMEM;

	/*
	 * set_tid_size contains the size of the set_tid array. Starting at
	 * the most nested currently active PID namespace it tells alloc_pid()
	 * which PID to set for a process in that most nested PID namespace
	 * up to set_tid_size PID namespaces. It does not have to set the PID
	 * for a process in all nested PID namespaces but set_tid_size must
	 * never be greater than the current ns->level + 1.
	 */
	if (set_tid_size > ns->level + 1)
		return ERR_PTR(-EINVAL);

	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
	if (!pid)
		return ERR_PTR(retval);

	tmp = ns;
	pid->level = ns->level;//pid��Ա��¼�½��̵�pid_ns��level��������ns�ı��

	for (i = ns->level; i >= 0; i--) {//�������и�pid_ns, pid�ṹ������Ҫ��¼ÿ��pid_ns�����pidֵ
		int tid = 0;

		if (set_tid_size) {
			tid = set_tid[ns->level - i];

			retval = -EINVAL;
			if (tid < 1 || tid >= pid_max)
				goto out_free;
			/*
			 * Also fail if a PID != 1 is requested and
			 * no PID 1 exists.
			 */
			if (tid != 1 && !tmp->child_reaper)
				goto out_free;
			retval = -EPERM;
			if (!ns_capable(tmp->user_ns, CAP_SYS_ADMIN))
				goto out_free;
			set_tid_size--;
		}

		idr_preload(GFP_KERNEL);
		spin_lock_irq(&pidmap_lock);

		if (tid) {
			nr = idr_alloc(&tmp->idr, NULL, tid,
				       tid + 1, GFP_ATOMIC);
			/*
			 * If ENOSPC is returned it means that the PID is
			 * alreay in use. Return EEXIST in that case.
			 */
			if (nr == -ENOSPC)
				nr = -EEXIST;
		} else {
			int pid_min = 1;
			/*
			 * init really needs pid 1, but after reaching the
			 * maximum wrap back to RESERVED_PIDS
			 */
			if (idr_get_cursor(&tmp->idr) > RESERVED_PIDS)
				pid_min = RESERVED_PIDS;

			/*
			 * Store a null pointer so find_pid_ns does not find
			 * a partially initialized PID (see below).
			 */
			nr = idr_alloc_cyclic(&tmp->idr, NULL, pid_min,
					      pid_max, GFP_ATOMIC);
		}
		spin_unlock_irq(&pidmap_lock);
		idr_preload_end();

		if (nr < 0) {
			retval = (nr == -ENOSPC) ? -EAGAIN : nr;
			goto out_free;
		}

		pid->numbers[i].nr = nr;
		pid->numbers[i].ns = tmp;
		tmp = tmp->parent;
	}

	/*
	 * ENOMEM is not the most obvious choice especially for the case
	 * where the child subreaper has already exited and the pid
	 * namespace denies the creation of any new processes. But ENOMEM
	 * is what we have exposed to userspace for a long time and it is
	 * documented behavior for pid namespaces. So we can't easily
	 * change it even if there were an error code better suited.
	 */
	retval = -ENOMEM;

	get_pid_ns(ns);
	refcount_set(&pid->count, 1);
	spin_lock_init(&pid->lock);
	for (type = 0; type < PIDTYPE_MAX; ++type)
		INIT_HLIST_HEAD(&pid->tasks[type]);

	init_waitqueue_head(&pid->wait_pidfd);
	INIT_HLIST_HEAD(&pid->inodes);

	upid = pid->numbers + ns->level;
	spin_lock_irq(&pidmap_lock);
	if (!(ns->pid_allocated & PIDNS_ADDING))
		goto out_unlock;
	for ( ; upid >= pid->numbers; --upid) {
		/* Make the PID visible to find_pid_ns. */
		idr_replace(&upid->ns->idr, pid, upid->nr);
		upid->ns->pid_allocated++;
	}
	spin_unlock_irq(&pidmap_lock);

	return pid;

out_unlock:
	spin_unlock_irq(&pidmap_lock);
	put_pid_ns(ns);

out_free:
	spin_lock_irq(&pidmap_lock);
	while (++i <= ns->level) {
		upid = pid->numbers + i;
		idr_remove(&upid->ns->idr, upid->nr);
	}

	/* On failure to allocate the first pid, reset the state */
	if (ns->pid_allocated == PIDNS_ADDING)
		idr_set_cursor(&ns->idr, 0);

	spin_unlock_irq(&pidmap_lock);

	kmem_cache_free(ns->pid_cachep, pid);
	return ERR_PTR(retval);
}
```

- ��ѭ���ڣ����ϵ���idr_alloc�ҵ���ǰnamespace�¿��е�nr��
	- `idr_alloc`���ƣ���redis���Ϸ���һ�����κ�ָ������Ķԣ�ָ��ָ��ڵ�
	- �൱�ڱ�����pid namespace�������ϵ�ÿ���ڵ㶼������һ��nr��pid


### �������ID��Ϣ

```c
//in copy_process func��
	p->pid = pid_nr(pid);   //��ȡglobal pid_ns�е�pid������0��
        �� ��
        init_task_pid(p, PIDTYPE_PID, pid);   //��struct pid �ṹ��ַ���浽������������
```


```c
static inline void
init_task_pid(struct task_struct *task, enum pid_type type, struct pid *pid)
{
	if (type == PIDTYPE_PID)
		task->thread_pid = pid;
	else
		task->signal->pids[type] = pid;
}
```

һЩ��ص����ݽṹ

1. task_struct�е���
```c
struct task_struct
{
        �� ��
	struct pid_link		pids[PIDTYPE_MAX];
        �� ��
}
```

2. PIDTYPE_MAX��ö��

```c
enum pid_type
{
	PIDTYPE_PID,
	PIDTYPE_PGID,
	PIDTYPE_SID,
	PIDTYPE_MAX,
	/* only valid to __task_pid_nr_ns() */
	__PIDTYPE_TGID
};
```

3. pid_link�ṹ��
```c
struct pid_link
{
	struct hlist_node node;
	struct pid *pid;
};
```

ע�⣺Linux�����ֽ��̺��̣߳�Ϊ�˼��ܷ����task struct�����ҵ���Ӧ��struct pid�����ܷ����struct pid�ܹ���������ʹ�ø�pid��task���ں������ struct pid_link ���������ID��Ӧ��struct pid �ṹ��ַ��

### ��ȡ���̵�PID value

Linux�ں��ṩ������׼API�Ի�ȡ����PID value

1. pid_nr()
��ȡ������pid
```c
static inline pid_t pid_nr(struct pid *pid)
{
	pid_t nr = 0;
	if (pid)
		nr = pid->numbers[0].nr;
	return nr;
}
```


2. pid_vnr()
��ȡ��ǰpid namespace��pid
```c
pid_t pid_vnr(struct pid *pid)
{
	return pid_nr_ns(pid, task_active_pid_ns(current));
}
EXPORT_SYMBOL_GPL(pid_vnr);
```

3. pid_nr_ns()
��ȡָ��namespace�е�pid value

```c
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)
{
	struct upid *upid;
	pid_t nr = 0;

	if (pid && ns->level <= pid->level) {
		upid = &pid->numbers[ns->level];
		if (upid->ns == ns)
			nr = upid->nr;
	}
	return nr;
}
EXPORT_SYMBOL_GPL(pid_nr_ns);
```
- level�Ĵ��ڷ����˺ܶ�

#### �û��ռ����getpid

```c
SYSCALL_DEFINE0(getpid)
{
	return task_tgid_vnr(current);
}
static inline pid_t task_tgid_vnr(struct task_struct *tsk)
{
	return __task_pid_nr_ns(tsk, __PIDTYPE_TGID, NULL);
}
pid_t __task_pid_nr_ns(struct task_struct *task, enum pid_type type,
	struct pid_namespace *ns)
{
	if (!ns)
		ns = task_active_pid_ns(current);
        �� ��
	nr = pid_nr_ns(rcu_dereference(task->pids[type].pid), ns);
	}
	rcu_read_unlock();

	return nr;
}
```
���յ��õ���`pid_nr_ns`�����ص��ǵ�ǰnamespace�е�pid��Ҳ����vpid

### ����pid��ȡpid_namespace

```c
static inline struct pid_namespace *ns_of_pid(struct pid *pid)
{
	struct pid_namespace *ns = NULL;
	if (pid)
		ns = pid->numbers[pid->level].ns;
	return ns;
}
```
ns_of_pid���Ը���pid��ȡpid_namespace���������ݽṹ���໥����

https://zhuanlan.zhihu.com/p/335171876
# UTS Namespace
- ��ϵͳ������namespace��

## �ļ�ϵͳ�ϵ�����

/proc/[pid]/ns/


# CGroup Namespace

���⻯���̶���cgroup���ӽ�

- ÿ��cgroup namespaceӵ�����Լ���cgroup��Ŀ¼����Ϊ`/proc/[pid]/cgroup`�ļ�����Ե�ַ�Ļ�����
- ��ͨ��`clone`��`unshare`����cgroup nsʱ����ǰ��cgroupĿ¼���Ϊ��ns�е�cgroup��Ŀ¼
- `/proc/[pid]/cgroup`��pathnameչʾ�˶�Ӧcgroup�µ����·��
- ���cgroupĿ¼�ڸ�Ŀ¼֮�⣬pathname����ʾ../
- 



# API
## ϵͳ���ýӿ�
### clone
### setns
### unshare

## �����нӿ�
nsenter