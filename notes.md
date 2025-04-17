# Source Code "Combing"

## Live Migration

- start a VM at the destination, copy memory pages, registers from source to destination
- live migration is divided into largely three steps: pre-copy memory migration, VM-state transfer, post-copy memory migration
  - pre-copy: memory pages are transferred concurrently from the source to the destination host; **pre-copy brownout**: contention on the memory subsystem, degraded performance
  - VM-state transfer; **blackout**: virtual machine is paused on the source and the state of the runtime is transferred to the destination to be resumed there.
  - post-copy: all pages left, and all dirty pages are re-transferred; **post-copy brownout**: application must wait for the memory to be transferred from the source before execution can continue.

## Live Migration Precopy

source VM diagram:

```cpp
// setting the migration rate and finally create the migraiton thread
void migration_connect(MigrationState *s, Error *error_in)
{
    if (error_in) {return;}
    // either s->parameters.max_postcopy_bandwidth; (for resumed postcopy)
    // or s->parameters.max_bandwidth; (for fresh new migration)
    migration_rate_set(rate_limit);

    // assuming no migrate_background_snapshot
    qemu_thread_create(&s->thread, MIGRATION_THREAD_SRC_MAIN,
                migration_thread, s, QEMU_THREAD_JOINABLE);
}

// includes pre-copy, vm state transfer, post-copy
static void *migration_thread(void *opaque)
{
    // just adding current thread to a global migration_threads list
    thread = migration_threads_add(MIGRATION_THREAD_SRC_MAIN,
                                   qemu_get_thread_id());

    // QEMU Machine Protocol (QMP) is a JSON-based protocol
    // using a "JSONWriter" to write qemu vm magic number, version number
    qemu_savevm_state_header(s->to_dst_file);

    // savevm_state is a global variable
    // iterate over a queue of handlers for ram, cpu, device ...
    // for pre-copy, only ram (I think, need to look at where these handlers are registered)
    // ret = se->ops->save_setup(f, se->opaque, errp);
    // see ram_save_setup below
    ret = qemu_savevm_state_setup(s->to_dst_file, &local_err);

    while (migration_is_active()) {
        if (urgent || !migration_rate_exceeded(s->to_dst_file)) {
            // transfers memory pages and calculates quota
            // keep track of a bitmap to track dirty pages
            MigIterateState iter_state = migration_iteration_run(s);
            // break if pending_size < s->threshold_size
            // skip means we are [starting to do] post copy(I think)
            // resume means another iteration (either pre/post copy)
            break if MIG_ITERATE_BREAK; continue if MIG_ITERATE_SKIP
        }

        urgent = migration_rate_limit();
    }

    static void migration_iteration_finish(MigrationState *s)
    {
        migration_bh_schedule(migration_cleanup_bh, s);
    }
}

// the hook function registered by ram handlers, there are other handlers for cpu, registers, devices ...
static int ram_save_setup(QEMUFile *f, void *opaque, Error **errp)
{
    static int ram_init_all(RAMState **rsp, Error **errp)
    {
        // init bitmaps for each RAMBlock
        // for pre-copy, all dirty bits are set to 1 (I think)
        ram_init_bitmaps(*rsp, errp)
    }
    RAMBLOCK_FOREACH_MIGRATABLE(block) {
        // writing block->idstr, block->used_length ...
        qemu_put_byte, qemu_put_buffer, qemu_put_be64
        // MIGRATION_CAPABILITY_MAPPED_RAM
        if (migrate_mapped_ram()) {
            mapped_ram_setup_ramblock(f, block);
        }
    }
    rdma_registration_start(f, RAM_CONTROL_SETUP);
    rdma_registration_stop(f, RAM_CONTROL_SETUP);

    if (migrate_multifd()) {
        multifd_ram_save_setup();
    }
    // legacy per-section sync ... (according to comment)
}
```

destination VM diagram:

```cpp
// create coroutine to server source side requests
void migration_incoming_process(void)
{
    Coroutine *co = qemu_coroutine_create(process_incoming_migration_co, NULL);
    qemu_coroutine_enter(co);
}

static void coroutine_fn
process_incoming_migration_co(void *opaque)
{
    int qemu_loadvm_state(QEMUFile *f)
    {
        qemu_loadvm_state_header(f);
        // again, for each cpu, ram, device handler:
        // ret = se->ops->load_setup(f, se->opaque, errp);
        // static SaveVMHandlers savevm_ram_handlers = {
        // .load_setup = ram_load_setup -> ramblock_recv_map_init (just init receive bitmap)
        qemu_loadvm_state_setup(f, &local_err)
        int qemu_loadvm_state_main(QEMUFile *f, MigrationIncomingState *mis)
        {
            case: QEMU_VM_SECTION_START || QEMU_VM_SECTION_FULL:
            // read section start, find savevm section, Validate version ...
                qemu_loadvm_section_start_full->vmstate_load
            case: QEMU_VM_SECTION_PART || QEMU_VM_SECTION_END:
                qemu_loadvm_section_part_end->vmstate_load
        }
    }
    migration_bh_schedule(process_incoming_migration_bh, mis);
}
```

## Live Migration Postcopy

src pov:

```cpp
void migration_connect(MigrationState *s, Error *error_in)
{
    if (error_in) {return;}
    // either s->parameters.max_postcopy_bandwidth; (for resumed postcopy)
    // or s->parameters.max_bandwidth; (for fresh new migration)
    migration_rate_set(rate_limit);

    // TODO: what exactly is a "return path"
    if (migrate_postcopy_ram() || migrate_return_path()) {
        // ms->rp_state.from_dst_file = qemu_file_get_return_path(ms->to_dst_file);
        // create thread: source_return_path_thread (Handles messages sent on the return path towards the source VM)
        open_return_path_on_source(s)
    }

    if (resume) {
        // TODO:
        /* Wakeup the main migration thread to do the recovery */
        migrate_set_state(&s->state, MIGRATION_STATUS_POSTCOPY_RECOVER_SETUP,
                          MIGRATION_STATUS_POSTCOPY_RECOVER);
    }

    // assuming no migrate_background_snapshot
    qemu_thread_create(&s->thread, MIGRATION_THREAD_SRC_MAIN,
                migration_thread, s, QEMU_THREAD_JOINABLE);
}


static void *migration_thread(void *opaque)
{
    // just adding current thread to a global migration_threads list
    thread = migration_threads_add(MIGRATION_THREAD_SRC_MAIN,
                                   qemu_get_thread_id());

    // QEMU Machine Protocol (QMP) is a JSON-based protocol
    // using a "JSONWriter" to write qemu vm magic number, version number
    qemu_savevm_state_header(s->to_dst_file);

    // If we opened the return path, we need to make sure 
    // dst has it opened as well.
    if (s->rp_state.rp_thread_created) {
        qemu_savevm_send_open_return_path(s->to_dst_file);
        qemu_savevm_send_ping(s->to_dst_file, 1);
    }

    // Tell the destination that we *might* want to do postcopy later;
    if (migrate_postcopy()) {
        qemu_savevm_send_postcopy_advise(s->to_dst_file);
    }

    // savevm_state is a global variable
    // iterate over a queue of handlers for ram, cpu, device ...
    // for pre-copy, only ram (I think, need to look at where these handlers are registered)
    // ret = se->ops->save_setup(f, se->opaque, errp);
    // see ram_save_setup below
    ret = qemu_savevm_state_setup(s->to_dst_file, &local_err);

    while (migration_is_active()) {
        if (urgent || !migration_rate_exceeded(s->to_dst_file)) {
            // whether we start post-copy is decided here
            // see postcopy_start below
            // for normal postcopy cases:
            // qemu_savevm_state_iterate(s->to_dst_file, in_postcopy);
            // se->ops->save_live_iterate->ram_save_iterate->ram_find_and_save_block
            MigIterateState iter_state = migration_iteration_run(s);
            break if MIG_ITERATE_BREAK; continue if MIG_ITERATE_SKIP
        }

        urgent = migration_rate_limit();
    }

    static void migration_iteration_finish(MigrationState *s)
    {
        migration_bh_schedule(migration_cleanup_bh, s);
    }
}

static int postcopy_start(MigrationState *ms, Error **errp)
{
    migration_stop_vm(ms, RUN_STATE_FINISH_MIGRATE);
    migration_switchover_start(ms, errp)
    // ...
    // TODO: switchover is a looooong process, re-check that part
    // this is officially switching to postcopy-active
    migrate_set_state(&ms->state, MIGRATION_STATUS_DEVICE,
                      MIGRATION_STATUS_POSTCOPY_ACTIVE);
}
```

dst pov:

```cpp
void hmp_migrate_incoming(Monitor *mon, const QDict *qdict)
    void qmp_migrate_incoming(const char *uri, bool has_channels,
                            MigrationChannelList *channels,
                            bool has_exit_on_error, bool exit_on_error,
                            Error **errp)
        static void qemu_start_incoming_migration(const char *uri, bool has_channels,
                                          MigrationChannelList *channels,
                                          Error **errp)
            if (addr->transport == MIGRATION_ADDRESS_TYPE_SOCKET) {
                socket_start_incoming_migration(saddr, errp);
            } else if (saddr->type == SOCKET_ADDRESS_TYPE_FD) {
                fd_start_incoming_migration(saddr->u.fd.str, errp);
            } else if (addr->transport == MIGRATION_ADDRESS_TYPE_RDMA) {
                rdma_start_incoming_migration(&addr->u.rdma, errp);
            } else {...}
```

```cpp
void socket_start_incoming_migration(SocketAddress *saddr,
                                     Error **errp)
{
    mis->transport_data = listener;
    mis->transport_cleanup = socket_incoming_migration_end;

    qio_net_listener_set_client_func_full(listener,
                                          socket_accept_incoming_migration,
                                          NULL, NULL,
                                          g_main_context_get_thread_default());
}

static void socket_accept_incoming_migration(QIONetListener *listener,
                                             QIOChannelSocket *cioc,
                                             gpointer opaque)
{
    void migration_channel_process_incoming(QIOChannel *ioc)
    {
        void migration_ioc_process_incoming(QIOChannel *ioc, Error **errp)
        {
            void migration_incoming_process(void)
            {
                Coroutine *co = qemu_coroutine_create(process_incoming_migration_co, NULL);
                qemu_coroutine_enter(co);
            }
        }
    }
}
```

```cpp
static void coroutine_fn
process_incoming_migration_co(void *opaque)
{
    int qemu_loadvm_state(QEMUFile *f)
    {
        qemu_loadvm_state_header(f);
        // again, for each cpu, ram, device handler:
        // ret = se->ops->load_setup(f, se->opaque, errp);
        // static SaveVMHandlers savevm_ram_handlers = {
        // .load_setup = ram_load_setup -> ramblock_recv_map_init (just init receive bitmap)
        qemu_loadvm_state_setup(f, &local_err)
        int qemu_loadvm_state_main(QEMUFile *f, MigrationIncomingState *mis)
        {
            case: QEMU_VM_SECTION_START || QEMU_VM_SECTION_FULL:
            // read section start, find savevm section, Validate version ...
                qemu_loadvm_section_start_full->vmstate_load
            case: QEMU_VM_SECTION_PART || QEMU_VM_SECTION_END:
                qemu_loadvm_section_part_end->vmstate_load
            case: QEMU_VM_COMMAND:
                loadvm_process_command(f)
                {
                    case MIG_CMD_PACKAGED:
                    // TODO: so here it looks like there is a loop here, what is happening??
                        loadvm_handle_cmd_packaged(mis)->qemu_loadvm_state_main(packf, mis);
                    // coincide with src pov
                    case MIG_CMD_POSTCOPY_ADVISE:
                        return loadvm_postcopy_handle_advise(mis, len);
                    case MIG_CMD_POSTCOPY_LISTEN:
                            loadvm_postcopy_handle_listen(mis) {
                                // TODO: looks like page fault handling to me??
                                postcopy_ram_incoming_setup->postcopy_thread_create(postcopy_ram_fault_thread)
                                postcopy_thread_create(postcopy_ram_listen_thread)
                            }
                    case MIG_CMD_POSTCOPY_RUN:
                        return loadvm_postcopy_handle_run(mis);
                }
        }
    }
    migration_bh_schedule(process_incoming_migration_bh, mis);
}
```
