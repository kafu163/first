# first
test



public final class FST<T> implements Accountable {

  private static final long BASE_RAM_BYTES_USED = RamUsageEstimator.shallowSizeOfInstance(FST.class);
  private static final long ARC_SHALLOW_RAM_BYTES_USED = RamUsageEstimator.shallowSizeOfInstance(Arc.class);

  /** Specifies allowed range of each int input label for
   *  this FST. */
  public static enum INPUT_TYPE {BYTE1, BYTE2, BYTE4};

  static final int BIT_FINAL_ARC = 1 << 0;
  static final int BIT_LAST_ARC = 1 << 1;
  static final int BIT_TARGET_NEXT = 1 << 2;

  // TODO: we can free up a bit if we can nuke this:
  static final int BIT_STOP_NODE = 1 << 3;

  /** This flag is set if the arc has an output. */
  public static final int BIT_ARC_HAS_OUTPUT = 1 << 4;

  static final int BIT_ARC_HAS_FINAL_OUTPUT = 1 << 5;

  // We use this as a marker (because this one flag is
  // illegal by itself ...):
  private static final byte ARCS_AS_FIXED_ARRAY = BIT_ARC_HAS_FINAL_OUTPUT;

  /**
   * @see #shouldExpand(Builder, Builder.UnCompiledNode)
   */
  static final int FIXED_ARRAY_SHALLOW_DISTANCE = 3; // 0 => only root node.

  /**
   * @see #shouldExpand(Builder, Builder.UnCompiledNode)
   */
  static final int FIXED_ARRAY_NUM_ARCS_SHALLOW = 5;

  /**
   * @see #shouldExpand(Builder, Builder.UnCompiledNode)
   */
  static final int FIXED_ARRAY_NUM_ARCS_DEEP = 10;

  // Increment version to change it
  private static final String FILE_FORMAT_NAME = "FST";
  private static final int VERSION_START = 0;

  /** Changed numBytesPerArc for array'd case from byte to int. */
  private static final int VERSION_INT_NUM_BYTES_PER_ARC = 1;

  /** Write BYTE2 labels as 2-byte short, not vInt. */
  private static final int VERSION_SHORT_BYTE2_LABELS = 2;

  /** Added optional packed format. */
  private static final int VERSION_PACKED = 3;

  /** Changed from int to vInt for encoding arc targets. 
   *  Also changed maxBytesPerArc from int to vInt in the array case. */
  private static final int VERSION_VINT_TARGET = 4;

  /** Don't store arcWithOutputCount anymore */
  private static final int VERSION_NO_NODE_ARC_COUNTS = 5;

  private static final int VERSION_PACKED_REMOVED = 6;

  private static final int VERSION_CURRENT = VERSION_PACKED_REMOVED;

  // Never serialized; just used to represent the virtual
  // final node w/ no arcs:
  private static final long FINAL_END_NODE = -1;

  // Never serialized; just used to represent the virtual
  // non-final node w/ no arcs:
  private static final long NON_FINAL_END_NODE = 0;

  /** If arc has this label then that arc is final/accepted */
  public static final int END_LABEL = -1;

  public final INPUT_TYPE inputType;

  // if non-null, this FST accepts the empty string and
  // produces this output
  T emptyOutput;

  /** A {@link BytesStore}, used during building, or during reading when
   *  the FST is very large (more than 1 GB).  If the FST is less than 1
   *  GB then bytesArray is set instead. */
  final BytesStore bytes;

  /** Used at read time when the FST fits into a single byte[]. */
  final byte[] bytesArray;

  private long startNode = -1;

  public final Outputs<T> outputs;

  private Arc<T> cachedRootArcs[];

  /** Represents a single arc. */
  public static final class Arc<T> {
    public int label;
    public T output;

    /** To node (ord or address) */
    public long target;

    byte flags;
    public T nextFinalOutput;

    // address (into the byte[]), or ord/address if label == END_LABEL
    long nextArc;

    /** Where the first arc in the array starts; only valid if
     *  bytesPerArc != 0 */
    public long posArcsStart;
    
    /** Non-zero if this arc is part of an array, which means all
     *  arcs for the node are encoded with a fixed number of bytes so
     *  that we can random access by index.  We do when there are enough
     *  arcs leaving one node.  It wastes some bytes but gives faster
     *  lookups. */
    public int bytesPerArc;

    /** Where we are in the array; only valid if bytesPerArc != 0. */
    public int arcIdx;

    /** How many arcs in the array; only valid if bytesPerArc != 0. */
    public int numArcs;

    /** Returns this */
    public Arc<T> copyFrom(Arc<T> other) {
      label = other.label;
      target = other.target;
      flags = other.flags;
      output = other.output;
      nextFinalOutput = other.nextFinalOutput;
      nextArc = other.nextArc;
      bytesPerArc = other.bytesPerArc;
      if (bytesPerArc != 0) {
        posArcsStart = other.posArcsStart;
        arcIdx = other.arcIdx;
        numArcs = other.numArcs;
      }
      return this;
    }
    
    boolean flag(int flag) {
      return FST.flag(flags, flag);
    }

    public boolean isLast() {
      return flag(BIT_LAST_ARC);
    }

    public boolean isFinal() {
      return flag(BIT_FINAL_ARC);
    }

    @Override
    public String toString() {
      StringBuilder b = new StringBuilder();
      b.append(" target=" + target);
      b.append(" label=0x" + Integer.toHexString(label));
      if (flag(BIT_FINAL_ARC)) {
        b.append(" final");
      }
      if (flag(BIT_LAST_ARC)) {
        b.append(" last");
      }
      if (flag(BIT_TARGET_NEXT)) {
        b.append(" targetNext");
      }
      if (flag(BIT_STOP_NODE)) {
        b.append(" stop");
      }
      if (flag(BIT_ARC_HAS_OUTPUT)) {
        b.append(" output=" + output);
      }
      if (flag(BIT_ARC_HAS_FINAL_OUTPUT)) {
        b.append(" nextFinalOutput=" + nextFinalOutput);
      }
      if (bytesPerArc != 0) {
        b.append(" arcArray(idx=" + arcIdx + " of " + numArcs + ")");
      }
      return b.toString();
    }
  };

  private static boolean flag(int flags, int bit) {
    return (flags & bit) != 0;
  }

  private final int version;

  // make a new empty FST, for building; Builder invokes
  // this ctor
  FST(INPUT_TYPE inputType, Outputs<T> outputs, int bytesPageBits) {
    this.inputType = inputType;
    this.outputs = outputs;
    version = VERSION_CURRENT;
    bytesArray = null;
    bytes = new BytesStore(bytesPageBits);
    // pad: ensure no node gets address 0 which is reserved to mean
    // the stop state w/ no arcs
    bytes.writeByte((byte) 0);

    emptyOutput = null;
  }

  public static final int DEFAULT_MAX_BLOCK_BITS = Constants.JRE_IS_64BIT ? 30 : 28;

  /** Load a previously saved FST. */
  public FST(DataInput in, Outputs<T> outputs) throws IOException {
    this(in, outputs, DEFAULT_MAX_BLOCK_BITS);
  }

  /** Load a previously saved FST; maxBlockBits allows you to
   *  control the size of the byte[] pages used to hold the FST bytes. */
  public FST(DataInput in, Outputs<T> outputs, int maxBlockBits) throws IOException {
    this.outputs = outputs;

    if (maxBlockBits < 1 || maxBlockBits > 30) {
      throw new IllegalArgumentException("maxBlockBits should be 1 .. 30; got " + maxBlockBits);
    }

    // NOTE: only reads most recent format; we don't have
    // back-compat promise for FSTs (they are experimental):
    version = CodecUtil.checkHeader(in, FILE_FORMAT_NAME, VERSION_PACKED, VERSION_CURRENT);
    if (version < VERSION_PACKED_REMOVED) {
      if (in.readByte() == 1) {
        throw new CorruptIndexException("Cannot read packed FSTs anymore", in);
      }
    }
    if (in.readByte() == 1) {
      // accepts empty string
      // 1 KB blocks:
      BytesStore emptyBytes = new BytesStore(10);
      int numBytes = in.readVInt();
      emptyBytes.copyBytes(in, numBytes);

      // De-serialize empty-string output:
      BytesReader reader = emptyBytes.getReverseReader();
      // NoOutputs uses 0 bytes when writing its output,
      // so we have to check here else BytesStore gets
      // angry:
      if (numBytes > 0) {
        reader.setPosition(numBytes-1);
      }
      emptyOutput = outputs.readFinalOutput(reader);
    } else {
      emptyOutput = null;
    }
    final byte t = in.readByte();
    switch(t) {
      case 0:
        inputType = INPUT_TYPE.BYTE1;
        break;
      case 1:
