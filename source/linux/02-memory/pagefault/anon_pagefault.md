# Pagefault

## pagefault入口

```c
 667 SYM_CODE_START_LOCAL_NOALIGN(el0_sync)
 668     kernel_entry 0
 669     mov x0, sp
 670     bl  el0_sync_handler
 671     b   ret_to_user
 672 SYM_CODE_END(el0_sync)
```

函数调用链：

```c
 53 static inline const struct fault_info *esr_to_fault_info(unsigned int esr)
 54 {
 55     return fault_info + (esr & ESR_ELx_FSC);
 56 }

el0_sync_handler
	el0_da
		do_mem_abort
			esr_to_fault_info  /* 从esr获取详细的fault信息，跳转到对应的callback处理   */
```

fault_info是一张表：

```c
653 static const struct fault_info fault_info[] = {
654     { do_bad,       SIGKILL, SI_KERNEL, "ttbr address size fault"   },
655     { do_bad,       SIGKILL, SI_KERNEL, "level 1 address size fault"    },
656     { do_bad,       SIGKILL, SI_KERNEL, "level 2 address size fault"    },
657     { do_bad,       SIGKILL, SI_KERNEL, "level 3 address size fault"    },
658     { do_translation_fault, SIGSEGV, SEGV_MAPERR,   "level 0 translation fault" },
659     { do_translation_fault, SIGSEGV, SEGV_MAPERR,   "level 1 translation fault" },
660     { do_translation_fault, SIGSEGV, SEGV_MAPERR,   "level 2 translation fault" },
661     { do_translation_fault, SIGSEGV, SEGV_MAPERR,   "level 3 translation fault" },
662     { do_bad,       SIGKILL, SI_KERNEL, "unknown 8"         },
663     { do_page_fault,    SIGSEGV, SEGV_ACCERR,   "level 1 access flag fault" },
664     { do_page_fault,    SIGSEGV, SEGV_ACCERR,   "level 2 access flag fault" },
665     { do_page_fault,    SIGSEGV, SEGV_ACCERR,   "level 3 access flag fault" },
666     { do_bad,       SIGKILL, SI_KERNEL, "unknown 12"            },
667     { do_page_fault,    SIGSEGV, SEGV_ACCERR,   "level 1 permission fault"  },668     { do_page_fault,    SIGSEGV, SEGV_ACCERR,   "level 2 permission fault"  },
669     { do_page_fault,    SIGSEGV, SEGV_ACCERR,   "level 3 permission fault"  },
670     { do_sea,       SIGBUS,  BUS_OBJERR,    "synchronous external abort"    },
671     { do_tag_check_fault,   SIGSEGV, SEGV_MTESERR,  "synchronous tag check fault"   },
672     { do_bad,       SIGKILL, SI_KERNEL, "unknown 18"            },
673     ......
};
```

这里我们先着重关注data_fault处理（do_page_fault）；

## do_page_fault

### do_anonymous_page

```c
3115 /*
3116  * We enter with non-exclusive mmap_sem (to exclude vma changes,
3117  * but allow concurrent faults), and pte mapped but not yet locked.
3118  * We return with mmap_sem still held, but pte unmapped and unlocked.
3119  */
3120 static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
3121 {
3122     struct vm_area_struct *vma = vmf->vma;
3123     struct mem_cgroup *memcg;
3124     struct page *page;
3125     vm_fault_t ret = 0;
3126     pte_t entry;
3127
3128     /* File mapping without ->vm_ops ? */
         /* 如果是共享的匿名映射，但是没有提供vma->vm_ops则返回VM_FAULT_SIGBUS错误 */
3129     if (vma->vm_flags & VM_SHARED)   
3130         return VM_FAULT_SIGBUS;
3131
3132     /*
3133      * Use pte_alloc() instead of pte_alloc_map().  We can't run
3134      * pte_offset_map() on pmds where a huge pmd might be created
3135      * from a different thread.
3136      *                                                                                                  3137      * pte_alloc_map() is safe to use under down_write(mmap_sem) or when                               3138      * parallel threads are excluded by other means.
3139      *                                                                                                  3140      * Here we only have down_read(mmap_sem).        
3141      */ 
         /* 如果pte不存在，则分配pte。建立pmd和pte的关系 */
3142     if (pte_alloc(vma->vm_mm, vmf->pmd, vmf->address))
3143         return VM_FAULT_OOM;
3144
3145     /* See the comment in pte_alloc_one_map() */
3146     if (unlikely(pmd_trans_unstable(vmf->pmd)))
3147         return 0;                                                                                       3148
3149     /* 如果是读操作触发的缺页且未禁止使用零页，则映射到全局零页 */
3150     if (!(vmf->flags & FAULT_FLAG_WRITE) &&
3151             !mm_forbids_zeropage(vma->vm_mm)) {
         /* 调用pte_mkspecial生成一个特殊页表项，映射到全局零页 */
3152         entry = pte_mkspecial(pfn_pte(my_zero_pfn(vmf->address),
3153                         vma->vm_page_prot));
         /* 根据pmd,address找到pte表对应的一个表项，并且lock住 */
3154         vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd, 
3155                 vmf->address, &vmf->ptl);
         /* 如果页表项不为空。我们第一次访问，不可能有值的，所以可能别的进程在使用此物理地址，跳unlock */
3156         if (!pte_none(*vmf->pte))
3157             goto unlock;
3158         ret = check_stable_address_space(vma->vm_mm);
3159         if (ret)
3160             goto unlock;
3161         /* Deliver the page fault to userland, check inside PT lock */
3162         if (userfaultfd_missing(vma)) {
3163             pte_unmap_unlock(vmf->pte, vmf->ptl);
3164             return handle_userfault(vmf, VM_UFFD_MISSING);
3165         }
3166         goto setpte;
3167     }
3168
3169     /* 分配一个匿名的anon_vma */
3170     if (unlikely(anon_vma_prepare(vma)))
3171         goto oom;
3172     page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
3173     if (!page)
3174         goto oom;
3175
3176     if (mem_cgroup_try_charge_delay(page, vma->vm_mm, GFP_KERNEL, &memcg,
3177                     false))
3178         goto oom_free_page;
3179
```

