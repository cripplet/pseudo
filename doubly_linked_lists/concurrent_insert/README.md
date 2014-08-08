Atomic Inserts for Doubly-Linked Lists
====

Given a doubly-linked node `n`, we wish to have the ability to append more nodes (i.e., a doubly-linked list starting at `hd` and ending at `tl`) to the end of `n`. We 
would like to call a function `n.insert(hd, tl)` such that:

* the operation is **atomic** and **deadlock-free**,
* only `n` and immediately adjacent nodes to `n` are locked in some manner (fine-grain locking),
* there is at most only a small memory overhead penalty due to locking structures

We will assume the following minimal data structure for a `Node` class (of which `n`, `hd`, and `tl` are instances):

```
class Node {
	public:
		/**
		 * list interface
		 */
		Node *get_next_safe() {
			Lock::acquire(this->next_l);
			Node *n = this->get_next_unsafe();
			Lock::release(this->next_l);
			return(n);
		}
		Node *get_prev_safe() { ... }
		void set_next_safe(Node *n) {
			Lock::acquire(this->next_l);
			this->set_next_unsafe(n);
			Lock::release(this->next_l);
		}
		void set_prev_safe(Node *n) { ... }
		void insert(Node *hd, Node *tl);

		/**
		 * note that these functions should not be called by the user, but must be public in order to implement Node::insert
		 */
		Node *get_next_unsafe() { return(this->next); }
		Node *get_prev_unsafe() { ... }
		void set_next_unsafe(Node *n) { this->next = n; }
		void set_prev_unsafe(Node *n) { ... }

	private:
		Node *next;
		Node *prev;
		Lock<std::atomic<bool>> next_l;
		Lock<std::atomic<bool>> prev_l;
}
```

Given this `Node` definition, we can implement `Node::insert` in the following manner:

```
public void Node::insert(Node *hd, Node *tl) {
	/* acquire locks */

	// this locks any further queries to this->next
	Lock::acquire(this->next_l);

	// this locks any further queries to next->prev
	Lock::acquire(this->get_next_unsafe()->prev_l);

	/* configure the list segment to be appended */

	hd->set_prev_unsafe(this);
	tl->set_next_unsafe(this->next);

	/* adding to the current list */

	// next->prev calls are currently blocked
	this->get_next_unsafe()->set_prev_unsafe(tl);

	// this->next calls are currently blocked
	this->set_next_unsafe(hd);

	/* release locks */

	// next->prev is now public (and will return tl)
	Lock::release(this->get_next_unsafe()->prev_l);

	// this->next is now public (and will return hd)
	Lock::release(this->next_l);	
}

```

Note the **order** of acquiring locks in `Node::insert` -- we are in effect extending a concurrent singly-linked list's hand-over-hand locking protocol to a 
doubly-linked list. The way in which we acquire the locks is strictly ordered, and symmetrical for all nodes, eliminating the possibility of a deadlock.

Because the `get` and `set` operations are trivial, we can use a spin-lock (using an atomic bool) to guard access, and in return have a low memory overhead (versus a 
more bloated mutex) -- for a large number of nodes, this can greatly reduce memory footprints.
