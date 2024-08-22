# Background

The syscalls design phylosophy is to provide a unified interface for the user to
access the system resources, which is object handle. This leads the object
becomes a fundamental concept in the COCONUT-SVSM kernel.

This document describes the object and object handle from both the user mode and
the COCONUT-SVSM kernel point of view.

# Key Data Structures

## Object

The object is a type of resource in the COCONUT-SVSM kernel, which is accessible
to the user mode, e.g., file, VM, VCPU, etc. So it doesn't have a fixed type,
which is better to be defined as a trait.

```Rust
trait Obj: Send + Sync {
    /// Open an object
    fn open(&self) -> Result<(), SvsmError> {
        Ok(())
    }

    /// Close an object
    fn close(&self) {}

    /// Convert to a virtual machine object if is supported.
    fn as_vm(&self) -> Option<&VmObj> {
        None
    }
    ...
}
```

A resource in the COCONUT-SVSM kernel which needs to be accessed by the user
mode should implement the `Obj` trait. E.g., a virtual machine object should
implement the `Obj` trait, to expose functionalities to the user mode via
syscalls.

```Rust
struct VmObj {
    id: VmId,
    ...
}

impl Obj for VmObj {
    fn open(&self) -> Result<(), SvsmError> {
        ...
    }

    fn close(&self) {
    }

    fn as_vm(&self) -> Option<&VmObj> {
        Some(&self)
    }
}

```

## Object Handle For User Mode

The object is exposed to the user mode via the syscalls by returning an object
handle, which is a unique identifier of the object. As the return value from the
syscall may be nagtive number as an error code, the object handle the user mode
received should be a positive number, which is defined as below:

```Rust
/// User mode object handle received from syscalls.
struct ObjHandle(i32);

impl ObjHandle {
    /// Create a new object handle with the raw number, which is returned from the syscalls.
    fn new(raw: i32) -> Result<Self, ObjHandleError> {
        // The raw id should not be a nagive number.
        if raw >= 0 {
            Ok(Self(raw))
        } else {
            Err(ObjHandleError::Invalid)
        }
    }

    /// Get the raw object handle id, which can be used as the input of the syscalls.
    fn raw(&self) -> i32 {
        self.0
    }
}
```

For a syscall class which consumes a perticular object handles, e.g., VMM
subsystem calls play with the object handle of virtual machine object, a
specific object handle can be defined as:

```Rust
struct VmObjHandle(ObjHandle);
```

To prevent an object handle being misused, the object handle doesn't implement
Copy/Clone trait, so that it can only be moved from one to another.

## Object Handle For COCONUT-SVSM Kernel

When the user mode is trying to open a perticular object via the syscalls, the
COCONUT-SVSM kernel creates an object handle, which is defined as below:

```Rust
structure ObjHandle {
    id: i32,
    obj: Arc<dyn Obj>,
}

impl ObjHandle {
    ///TODO
    pub fn new(id: i32, obj: Arc<dyn Obj>) -> Result<Self, SvsmError> {
        obj.open()?;
        Ok(Self { id, obj })
    }

    /// Get the ObjHandle id.
    pub fn id(&self) -> i32 {
        self.id
    }
}

impl Drop for ObjHandle {
    fn drop(&mut self) {
        self.obj.close();
    }
}

impl Obj for ObjHandle {
    fn as_vm(&self) -> Option<&VmObj> {
        self.obj.as_vm()
    }
}
```

The ObjHandle contains an unique id allocated by a global ObjIDAllocator, and
the opened object which implements the Obj trait. When a new ObjHandle is
created, the object is opened by calling the `open` method. And when the
ObjHandle is dropped, the object is closed by calling the `close` method. The
unique id of this ObjHandle will be returned as the user mode ObjHandle, and the
subsequent syscalls can use this id to access this object.

This requires the COCONUT-SVSM kernel to take below responsibilities:

- The COCONUT-SVSM kernel should manage the object handle's lifecycle properly.
  The object handle should be released when the user mode closes the object, or
  the user mode is terminated without closing.
- The COCONUT-SVSM kernel should prevent one user mode thread misusing the
  object handle opened by another.

To achieve the above goals, the created object handles should be associated with
the task which creates it. The task structure is extended to hold the ObjHandles
created by this thread.

```Rust
pub struct Task {
    ...

    /// Object handles created by this task
    obj_handles: RWLock<BTreeMap<i32, Arc<ObjHandle>>>,
}
```

The task structure will provide 3 new functions:

- add_obj_handle(&self, obj_handle: Arc<ObjHandle>) -> Result<(), SvsmError> is
  to add the object handle to the BTreeMap with the object handle id as the key.
  This will be used by the syscalls which open an object.
- get_obj_handle(&self, id: i32) -> Result<Arc<ObjHandle>, SvsmError> is to get
  the object handle from the BTreeMap by cloning an Arc<ObjHandle>. This will be
  used by the syscalls which access an object.
- remove_obj_handle(&self, id: i32) -> Result<Arc<ObjHandle>, SvsmError> is to
  remove the object handle from the BTreeMap. This wil be used by the syscalls
  which close an object.

When a task is terminated while it still has opened objects, these objects will
be closed automatically when the ObjHandles are dropped.

# User Mode Open An Object

The user mode can open a perticular object via the syscalls. For example,
VM_OPEN syscall is used to open a virtual machine object. The COCONUT-SVSM
kernel provides a new function obj_open() to facilitate the user mode to open an
object.

```Rust
pub fn obj_open(obj: Arc<dyn Obj>) -> Result<i32, SvsmError> {
    let id = OBJ_ID_ALLOCATOR.next_id();
    current_task()
        .add_obj_handle(Arc::new(ObjHandle::new(id, obj)?))
        .map(|_| id)
}
```

The obj_open() takes the Arc<dyn Obj> as an input, which represents the
perticular object to be opened. After allocates a unique id, it creates a new
ObjHandle with the object opened and add this ObjHandle to the current task with
the add_obj_handle() method. The unique id of the ObjHandle is returned to the
user mode as the ObjHandle.

```Rust
impl Task {
    pub fn add_obj_handle(&self, obj_handle: Arc<ObjHandle>) -> Result<(), SvsmError> {
        if let Entry::Vacant(entry) = self.obj_handles.lock_write().entry(obj_handle.id()) {
            entry.insert(obj_handle);
            Ok(())
        } else {
            Err(ObjError::Exist.into())
        }
    }
}
```

# User Mode Get An Object

Certain syscalls can access the objects by taking the object id as an input. For
example, VM_CAPABILITIES syscall takes an object id as input, and returns the
requested capability of the virtual machine object. The COCONUT-SVSM kernel can
retrieve the object handle by the object id, to access the corresponding object.

```Rust
pub fn obj_handle(id: i32) -> Result<Arc<ObjHandle>, SvsmError> {
    current_task().get_obj_handle(id)
}

pub fn sys_vm_capabilities(obj_id: i32, idx: u32) -> u64 {
    if let Ok(handle) = obj_handle(obj_id) {
        if let Some(vm) = handle.as_vm() {
            vm.capabilities(idx)
        } else {
            0
        }
    } else {
        0
    }
}
```

The obj_handle() get an Arc<ObjHandle> by the object id from the current task,
which increase the reference count of the object handle. Once the syscall
complete the usage of the object handle, the reference count will be decreased,
like the example of sys_vm_capabilities() shows.

```Rust
impl Task {
    pub fn get_obj_handle(&self, id: i32) -> Result<ObjHandlePointer, SvsmError> {
        self.obj_handles.lock_read().get(&id).cloned().map_or_else(
            || Err(ObjError::NotFound.into()),
            |handle| Ok(handle),
        )
    }
}
```

## Map An Object

The MMAP syscal is able to mmap a perticular object to the user mode address
space, if this object provides a mappable FileHandle.

```Rust
trait Obj: Send + Sync {
    /// Open an object
    fn open(&self) -> Result<(), SvsmError> {
        Ok(())
    }

    /// Close an object
    fn close(&self) {}

    /// Convert to a virtual machine object if is supported.
    fn as_vm(&self) -> Option<&VmObj> {
        None
    }

    /// Get a mappable file handle if the object is mappable.
    fn mappable(&self) -> Option<&FileHandle> {
        None
    }
}

```

The retrived Arc<ObjHandle> should not be released until the user mode unmaps
the object. To achieve this, the Arc<ObjHandle> should be associated with the
mapping. As the FileHandle is used to create VMFileMapping, the VMFileMapping is
extended as below:

```Rust
pub enum MapTarget {
    Anon,
    File(&'a FileHandle),
    ObjId(i32),
}

pub struct VMFileMapping {
    ...

    /// The corresponding ObjHandlePointer of the mapped pages. Saving the
    /// object handle here to make sure the ObjHandle is valid until VMFileMapping
    /// is dropped.
    obj_handle: Arc<ObjHandle>,
}

impl VMFileMapping {
    /// Create a new ['VMFileMapping'] for a file. The file provides the backing
    /// pages for the file contents.
    ///
    /// # Arguments
    ///
    /// * 'target' - The target to create the mapping for. It would be a file reference
    ///   or an object which can support file mapping.
    ///
    /// * 'offset' - The offset from the start of the file to map. This must be
    ///   align to PAGE_SIZE.
    ///
    /// * 'size' - The number of bytes to map starting from the offset. This
    ///   must be a multiple of PAGE_SIZE.
    ///
    /// # Returns
    ///
    /// Initialized mapping on success, Err(SvsmError::Mem) on error
    pub fn new(
        target: MapTarget<'_>,
        offset: usize,
        size: usize,
        flags: VMFileMappingFlags,
    ) -> Result<Self, SvsmError> {
        match target {
            MapTarget::Anon => Err(SvsmError::Mem),
            MapTarget::File(file) => Self::create(file, None, offset, size, flags),
            MapTarget::ObjId(obj_id) => {
                let obj_handle = current_task().get_obj_handle(obj_id)?;
                let file = obj_handle
                    .mappable()
                    .and_then(|file| {
                        if file.size() < offset + size {
                            None
                        } else {
                            Some(file)
                        }
                    })
                    .ok_or(SvsmError::Mem)?;
                Self::create(file, Some(obj_handle.clone()), offset, size, flags)
            }
        }
    }

    fn create(
        file: &FileHandle,
        obj_handle: Option<Arc<ObjHandle>>,
        offset: usize,
        size: usize,
        flags: VMFileMappingFlags,
    ) -> Result<Self, SvsmError> {
        ...
    }
}
```

Enum MapTarget is created to represent anonymous mapping, file mapping, and
object mapping. The VMFileMapping only supports MapTarget::File(&FileHandle) and
MapTarget::ObjId(i32), but not the MapTarget::Anon. The VMFileMapping is
extended to hold the Arc<ObjHandle> if maps an object. The Arc<ObjHandle> is
cloned and saved in the VMFileMapping, to make sure the ObjHandle is valid until
the VMFileMapping is dropped when munmaps this object or the task is terminated.

# User Mode Close An Object

The CLOSE syscall can close an object from the user space, with taking the
object id as the input parameter. The COCONUT-SVSM kernel can remove the
ObjHandle from the current task by the object id.

```Rust
pub fn sys_close(obj_id: i32) -> i32 {
    // According to syscall ABI/API spec, close always returns 0 even
    // if called with an invalid handle
    let _ = obj_close(obj_id);
    0
}

pub fn obj_close(id: i32) -> Result<Arc<ObjHandle>, SvsmError> {
    current_task().remove_obj_handle(id)
}

impl Task {
    pub fn remove_obj_handle(&self, id: i32) -> Result<Arc<ObjHandle>, SvsmError> {
        let mut obj_handles = self.obj_handles.lock_write();

        if let Some(obj_handle) = obj_handles.get(&id) {
            // Cannot remove the ObjHandle if it is in-use
            if Arc::strong_count(obj_handle) > 1 {
                return Err(ObjError::Busy.into());
            }
        }

        obj_handles.remove(&id).ok_or(ObjError::NotFound.into())
    }
}

```

Before remove an ObjHandle from its obj_handles BTreeMap, its reference count
should be checked. If the reference count is greater than 1, it means the object
is still in use, either the user mode is using some syscalls with this object id
or the user mode has mapped this object wihout unmaping. In this case, the
COCONUT-SVSM kernel should not remove this ObjHandle. Once the ObjHandle is
successfully removed from the obj_handles BTreeMap, it will be dropped and the
object is closed.
