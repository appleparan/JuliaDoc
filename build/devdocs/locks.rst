.. currentmodule:: Base

****************************************************
Proper maintenance and care of multi-threading locks
****************************************************

The following strategies are used to ensure that the code is dead-lock free
(generally by addressing the 4th Coffman condition: circular wait).

 1. structure code such that only one lock will need to be acquired at a time

 2. always acquire shared locks in the same order, as given by the table below

 3. avoid constructs that expect to need unrestricted recursion

Below are all of the locks that exist in the system and the mechanisms
for using them that avoid the potential for deadlocks (no Ostrich algorithm allowed here):

The following are definitely leaf locks (level 1), and must not try to acquire any other lock:

   * safepoint

       Note that this lock is acquired implicitly by ``JL_LOCK`` and ``JL_UNLOCK``.
       use the ``_NOGC`` variants to avoid that for level 1 locks.

       While holding this lock, the code must not do any allocation or hit any safepoints.
       Note that there are safepoints when doing allocation, enabling / disabling GC,
       entering / restoring exception frames, and taking / releasing locks.

   * shared_map
   * finalizers
   * pagealloc
   * gc_perm_lock
   * flisp

       flisp itself is already threadsafe, this lock only protects the ``jl_ast_context_list_t`` pool


The following is a leaf lock (level 2), and only acquires level 1 locks (safepoint) internally:

   * dump (this lock may be possible to eliminate by storing the state on the stack)
   * typecache

The following are level 3 locks, which can only acquire level 1 or level 2 locks internally:

   * Method->writelock

     but note that this is violated by staged functions!

The following is a level 4 lock, which can only recurse to acquire level 1, 2, or 3 locks:

   * MethodTable->writelock

     but note that this is violated by staged functions!

The following is a proposed level 5 lock, which can only recurse to acquire locks at lower levels:

   * staged

       this theoretical lock would create a priority inversion from the `method->writelock` (level 3),
       but only prohibiting running any staging function in parallel
       (thus allowing temporary release of the MethodTable and Method locks)

The following is a level 6 lock, which can only recurse to acquire locks at lower levels:

   * codegen

The following is an almost root lock (level end-1), meaning only the root look may be held when trying to acquire it:

   * typeinf

       this one is perhaps one of the most tricky ones, since type-inference can be invoked from many points

The following is the root lock, meaning no other lock shall be held when trying to acquire it:

   * toplevel

       this should be held while attempting a top-level action (such as making a new type or defining a new method):
       trying to obtain this lock inside a staged function will cause a deadlock condition!

       additionally, it's unclear if *any* code can safely run in parallel with an arbitrary toplevel expression,
       so it may require all threads to get to a safepoint first


The following locks are broken:
-------------------------------

* toplevel

    doesn't exist right now

    fix: create it

* codegen

    recursive (through ``static_eval``), but caller might also be holding locks (due to staged functions)

    other issues?

    fix: prohibit codegen while holding any other lock (possibly by checking ``ptls->current_task->locks.len != 0`` & explicitly check the locks that are OK to hold simultaneously)?

* typeinf

    not certain of whether there are issues here or what they are. staging functions, of course, are a source of deadlocks here.

    fix: unknown

* staged

    possible solution to prevent staged functions from causing deadlock.

    this theoretical lock would create a priority inversion such that the Method and MethodTable write locks could be released
    by ensuring that no staging functions can run in parallel allow this level 5 lock to protect staged function conflicts (a level 3 operation)

    fix: create it

* typecache

    this only protects cache lookup and insertion but it doesn't make the lookup-and-construct atomic

    fix: lock for ``apply_type`` / global (level 2?)


* dump

    may be able to eliminate this by making the state thread-local

    might also need to ensure that incremental deserialize has a toplevel expression lock

