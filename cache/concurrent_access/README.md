Concurrent Access to Cache
====

The concept of a cache is a simple one -- have data that is used most recently load the fastest. But what happens when concurrent requests are made to the cache for 
data? How does the cache deal with adding and removing entries?

A key characteristic of most caches is that **the number of cache entries remains constant** -- we will statically allocate some fixed-size memory space for the cache at 
program startup time, and then move cached entries in and out of the cache during process time.

```
template <class T>
class Cache {
	public:
		void acquire(T* arg) {
			Lock::acquire(this->access_l.at(this->hash(arg)));
			if(!this->in(T)) {
				Lock::release(this->access_l.at(this->hash(arg)));
				this->allocate(T);
			}
		}

	private:
		std::map<size_t, T*> cache;
		size_t n_entries;
		std::vector<Lock<std::mutex>> access_l;
		Lock<std::mutex> alloc_l;

		// add new cache entry to the cache
		void allocate(T* arg) {
			Lock::acquire(this->alloc_l);
			Lock::acquire(this->access_l.at(this->hash(arg)));
			// possible the block has already been inserted into the cache by a competing call
			if(this->in(T)) {
				Lock::release(this->alloc_l);
				return;
			}
			if(n_entries < this->cache.size()) {
				size_t target = 0;
				size_t heuristic = 0;
				for(std::map<size_t, T*>::iterator it = this->cache.begin(); it != this->cache.end(); ++it) {
					size_t index = this->hash(it->second);
					if(index != this->hash(arg)) { Lock::acquire(this->access_l(index)); }
					if((this->heuristic(it->second) < heuristic) || (heuristic == 0)) {
						heuristic = this->heuristic(it->second);
						target = it->first;
					}
					if(index != this->hash(arg)) { Lock::release(this->access_l(index)); }
				}
				target->unload();
				this->cache.erase(target);
				
			}
			
			arg->load();
			Lock::release(this->alloc_l);
		}

		// the appropriate cache line must be held before calling this function
		bool in(T* arg) { return(arg == this->cache.at(this->hash(arg))); }

		/**
		 * virtual fields
		 */

		// the selection policy
		// this->l must be acquired before calling this function
		virtual T* select();

		// returns a cache line index
		virtual size_t hash(T* arg);

};
```
