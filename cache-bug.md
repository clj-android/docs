È33      private static final CfOptions CF_OPTIONS = new CfOptions();
      34      private static final DexOptions DEX_OPTIONS = new DexOptions();
      35
      36 +    /**
      37 +     * Tracks the class name currently being defined, per thread.
      38 +     *
      39 +     * <p>When {@link #defineClass} is in progress, the InMemoryDexClassLoader
      40 +     * delegates to this classloader (its parent) via {@code loadClass}. Without
      41 +     * this guard, the parent would find the OLD class in {@code classCache} and
      42 +     * return it — the new DEX bytecodes would never be consulted. By tracking
      43 +     * which class is being redefined, {@link #loadClass} can throw
      44 +     * {@code ClassNotFoundException} for that specific class, forcing the
      45 +     * InMemoryDexClassLoader to fall through to its own {@code findClass} and
      46 +     * load from the fresh DEX bytes.</p>
      47 +     */
      48 +    private static final ThreadLocal<String> classBeingDefined = new ThreadLocal<>();
      49 +
      50      static {
      51          CF_OPTIONS.strictNameCheck = false;
      52          DEX_OPTIONS.minSdkVersion = 26;
     ...
      71      @Override
      72      public Class defineClass(String name, byte[] bytes, Object srcForm) {
      73          try {
      74 +            // Mark this class as being redefined so loadClass won't return
      75 +            // stale versions from the cache or parent classloader chain.
      76 +            classBeingDefined.set(name);
      77 +
      78 +            // Remove stale cache entry before loading the new version.
      79 +            classCache.remove(name);
      80 +
      81              // Translate JVM bytecode to DEX using the dx library
      82              DxContext dxContext = new DxContext();
      83              DexFile dexFile = new DexFile(DEX_OPTIONS);
     ...
      90              dexFile.writeTo(baos, null, false);
      91              byte[] dexBytes = baos.toByteArray();
      92
      72 -            // Load via InMemoryDexClassLoader (API 26+, no temp files needed)
      93 +            // Load via InMemoryDexClassLoader (API 26+, no temp files needed).
      94 +            // The dexLoader uses this classloader as its parent for delegation.
      95 +            // Our loadClass override ensures the class being defined is NOT found
      96 +            // via the parent, forcing dexLoader to load from the fresh DEX bytes.
      97              ByteBuffer dexBuffer = ByteBuffer.wrap(dexBytes);
      98              InMemoryDexClassLoader dexLoader = new InMemoryDexClassLoader(dexBuffer, this);
      99
     ...
      115          } catch (IOException e) {
      116              Log.e(TAG, "I/O error translating class to DEX: " + name, e);
      117              throw new RuntimeException("Failed to define class " + name + " on Android", e);
      118 +        } finally {
      119 +            classBeingDefined.remove();
      120          }
      121      }
      122
      123 +    /**
      124 +     * Prevents returning stale class definitions during function redefinition.
      125 +     *
      126 +     * <p>When InMemoryDexClassLoader.loadClass delegates to this parent
      127 +     * classloader, we must NOT return the old version of a class that is
      128 +     * currently being redefined. Throwing ClassNotFoundException forces
      129 +     * InMemoryDexClassLoader to fall through to its own findClass(), which
      130 +     * loads from the fresh DEX bytes containing the new function body.</p>
      131 +     *
      132 +     * <p>This handles both cases:</p>
      133 +     * <ul>
      134 +     *   <li>Old class in classCache (from a previous REPL definition)</li>
      135 +     *   <li>Old class in parent classloader chain (from AOT compilation)</li>
      136 +     * </ul>
      137 +     */
      138 +    @Override
      139 +    protected synchronized Class<?> loadClass(String name, boolean resolve)
      140 +            throws ClassNotFoundException {
      141 +        if (name.equals(classBeingDefined.get())) {
      142 +            throw new ClassNotFoundException(name);
      143 +        }
      144 +        return super.loadClass(name, resolve);
      145 +    }
      146 +
      147      private static String classNameToPath(String name) {
      148          return name.replace('.', '/') + ".class";
      149      }


