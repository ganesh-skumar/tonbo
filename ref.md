starts at dbinner.ingest() db/mod.rs 66

## WAL
codex:
let mut sender = self.inner.clone_sender(); does not write to disk by itself.

clone_sender() returns a cloned mpsc::Sender<writer::WriterMsg> for the WAL writer task’s bounded queue. The consumer is the async loop in src/wal/writer.rs (run_writer_loop, spawned by writer::spawn_writer). See src/wal/mod.rs:458 and src/wal/writer.rs:123.
Where the WAL message goes

sender.send(msg).await enqueues writer::WriterMsg::Enqueue { command: WalCommand::..., ... } into that queue. See src/wal/mod.rs:549 and src/wal/writer.rs:48.
The writer task (consumer) dequeues it in run_writer_loop and then encodes/writes it. See src/wal/writer.rs:177 and the WriterMsg::Enqueue match arm at src/wal/writer.rs:227.
Where it writes on storage (disk/S3/etc.)

The writer task writes frames into WAL segment files via WalStorage.
Segments live under WalConfig.dir on WalConfig.segment_backend (a DynFs). The storage facade is created at WAL enable time:

WalStorage::new(Arc::clone(&cfg.segment_backend), cfg.dir.clone()) in enable_wal. See src/wal/mod.rs:953.
A segment path is cfg.dir / format!("wal-{seq:020}.tonwal"). See src/wal/storage.rs:97.
WalStorage::open_segment() opens/creates that file with fusio OpenOptions in append mode (write(true), create(true), truncate(false)), 

so subsequent writes append frames. See src/wal/storage.rs:45 and src/wal/storage.rs:81.
So: clone_sender() → enqueue to writer queue → writer loop consumes → WalStorage appends to cfg.dir/wal-000000000000000000XX.tonwal on whatever backend cfg.segment_backend is (local disk by default).