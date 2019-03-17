---
title: "Atomic operations"
menuTitle: "Atomic operations"
date: 2018-06-27T17:17:25+02:00
weight: 15
---

The effect of [virtualizing interrupt protection]({{%relref
"dovetail/pipeline/porting/arch.md" %}}) must be reversed for atomic
helpers in *asm-generic/atomic.h*, *asm-generic/bitops/atomic.h* and
*asm-generic/cmpxchg-local.h*, so that no interrupt can preempt their
execution, regardless of the stage their caller live on.

This is required to keep those helpers usable on data which might be
accessed from both stages.

The usual way to revert such virtualization consists of delimiting the
protected section with `hard_local_irq_save()`,
`hard_local_irq_restore()` calls, in replacement for
`local_irq_save()`, `local_irq_restore()` respectively.

> Restoring strict serialization for operations on generic atomic counters

```
--- a/include/asm-generic/atomic.h
+++ b/include/asm-generic/atomic.h
@@ -80,9 +80,9 @@ static inline void atomic_##op(int i, atomic_t *v)			\
 {									\
 	unsigned long flags;						\
 									\
-	raw_local_irq_save(flags);					\
+	flags = hard_local_irq_save();					\
 	v->counter = v->counter c_op i;					\
-	raw_local_irq_restore(flags);					\
+	hard_local_irq_restore(flags);					\
 }
 
 #define ATOMIC_OP_RETURN(op, c_op)					\
@@ -91,9 +91,9 @@ static inline int atomic_##op##_return(int i, atomic_t *v)		\
 	unsigned long flags;						\
 	int ret;							\
 									\
-	raw_local_irq_save(flags);					\
+	flags = hard_local_irq_save();					\
 	ret = (v->counter = v->counter c_op i);				\
-	raw_local_irq_restore(flags);					\
+	hard_local_irq_restore(flags);					\
 									\
 	return ret;							\
 }
@@ -104,10 +104,10 @@ static inline int atomic_fetch_##op(int i, atomic_t *v)			\
 	unsigned long flags;						\
 	int ret;							\
 									\
-	raw_local_irq_save(flags);					\
+	flags = hard_local_irq_save();					\
 	ret = v->counter;						\
 	v->counter = v->counter c_op i;					\
-	raw_local_irq_restore(flags);					\
+	hard_local_irq_restore(flags);					\
 									\
 	return ret;							\
 }
```

Likewise, such operations may exist in architecture-specific code,
overriding their generic definitions. For instance, the ARM port
defines its own version of atomic operations for which real interrupt
protection has to be reinstated:

> Restoring strict serialization for operations on atomic counters for ARM

```
--- a/arch/arm/include/asm/atomic.h
+++ b/arch/arm/include/asm/atomic.h
@@ -168,9 +168,9 @@ static inline void atomic_##op(int i, atomic_t *v)			\
 {									\
 	unsigned long flags;						\
 									\
-	raw_local_irq_save(flags);					\
+	flags = hard_local_irq_save();					\
 	v->counter c_op i;						\
-	raw_local_irq_restore(flags);					\
+	hard_local_irq_restore(flags);					\
 }									\
 
 #define ATOMIC_OP_RETURN(op, c_op, asm_op)				\
@@ -179,10 +179,10 @@ static inline int atomic_##op##_return(int i, atomic_t *v)		\
 	unsigned long flags;						\
 	int val;							\
 									\
-	raw_local_irq_save(flags);					\
+	flags = hard_local_irq_save();					\
 	v->counter c_op i;						\
 	val = v->counter;						\
-	raw_local_irq_restore(flags);					\
+	hard_local_irq_restore(flags);					\
 									\
 	return val;							\
 }
@@ -193,10 +193,10 @@ static inline int atomic_fetch_##op(int i, atomic_t *v)			\
 	unsigned long flags;						\
 	int val;							\
 									\
-	raw_local_irq_save(flags);					\
+	flags = hard_local_irq_save();					\
 	val = v->counter;						\
 	v->counter c_op i;						\
-	raw_local_irq_restore(flags);					\
+	hard_local_irq_restore(flags);					\
 									\
 	return val;							\
 }
@@ -206,11 +206,11 @@ static inline int atomic_cmpxchg(atomic_t *v, int old, int new)
 	int ret;
 	unsigned long flags;
 
-	raw_local_irq_save(flags);
+	flags = hard_local_irq_save();
 	ret = v->counter;
 	if (likely(ret == old))
 		v->counter = new;
-	raw_local_irq_restore(flags);
+	hard_local_irq_restore(flags);
 
 	return ret;
}
```
