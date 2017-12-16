# remote-sync
Synchronize remotely stored directories.

## Description

The tools in this package allow you to sync the contents of 2 possibly remotely stored root directories, A and B, so that they will become identical. For the sake of preventing back-and-forth and repeatedly scanning the original root directories, this syncing is done indirectly through *syncmaps*.

A syncmap is a map of a root directory. This map is a directory itself, and must contain a **root.map** file, containing data about each of the files in the original directory, and also data about how the synchronization process for each of those files is going. Also, the syncmap dir may contain dirs and/or files -- all of whose paths should be referenced by the **root.map** file. Any such files will be specially typed files (by default, and as referred to in this document, these are all .txt files) or checksum files (currently supported: .hash), and in fact any .txt or .hash files which existed in the directory that the syncmap is a map of will exist in the input syncmap. In terms of usage, you can think of there being *input syncmaps*, which are direct maps of a root directory. After 2 input syncmaps are compared, 2 *mod syncmaps* are created, one per input syncmap. The mod syncmaps include everything in the original syncmaps, but also contain possibly altered .txt and .hash files (the .txt files would be manually changed by the user), and will reference files that are not in the original root directories but should be -- and have been copied into special *mod directories* created alongside the mod syncmaps. Files in the mod directories should be moved over into the original root directories in order to make those root directories exact mirrors of each other.

## Usage example

A and B are two directories whose contents mostly mirror each other but may each have differences. To turn them into exact mirrors of each other, do the following steps:

1. Run: `python create_syncmap.py A`
2. Run: `python create_syncmap.py B`
3. Run: `python compare_syncmaps.py A_syncmap B_syncmap`

 (This creates these directories: A\_mod\_syncmap, A\_mod\_dir, B\_mod\_syncmap, and B\_mod\_dir. A\_mod\_dir shows changes that should be made to A; B\_mod\_dir shows changes that should be made to B. The mod syncmaps shows what the original syncmaps would be like after applying these changes. Any files with unreconcilable differences don't show up in the mod dirs, but do show up in the mod syncmaps.)

4. As long as you get warnings:

 i. Make changes to unreconcilable differences between A\_mod\_syncmap and B\_mod\_syncmap

 ii. Run: `python compare_syncmaps.py A_mod_syncmap B_mod_syncmap --fix`

 iii. Repeat these steps

 (Running *fix mode* with **--fix** or **-f** verifies that the mod syncmaps are the same, without creating any new layer of syncmaps. Also, if you successfully fixed any unreconcilable differences manually, using fix mode copies those manual changes you made to the mod syncmaps into the mod dirs.)

5. Delete A\_mod\_syncmap and B\_mod\_syncmap
6. Copy contents of A\_mod\_dir to A
7. Copy contents of B\_mod\_dir to B

(Note that you can pass the **--del\_on\_success** or **-d** argument to to **compare\_syncmaps.py**. In the event that there are no warnings, this will result in the mod syncmap directories being deleted automatically.)

## Usage example, in greater detail

1. Run `python create\_syncmap.py A` to create a syncmap of A, which we will refer to as A\_syncmap. Any .txt or .hash files are copied over into the A\_syncmap dir.

2. Run `python create\_syncmap.py B` to create a syncmap of B: B\_syncmap. It doesn't matter if A and B are on separate computers, as long as their syncmaps are both accessible for the next step.

3. Run `python compare\_syncmaps.py A\_syncmap B\_syncmap`. It will generate A\_mod\_syncmap and B\_mod\_syncmap, alongside the original directories. A\_mod\_syncmap is a copy of A\_syncmap, but contains references to new files to show what needs to be copied over into A to make it a mirror of B (after both A\_mod\_syncmap and B\_mod\_syncmap are applied back into the original root directories). Additionally, A\_mod\_dir is created alongside A\_mod\_syncmap, and it contains any of the new files that should be copied from B to A. B\_mod\_syncmap is the same thing, only to shows the results of an attempt to make B become a mirror of A. Note that any .txt or .hash files which have irreconcilable differences will also be created in the mod syncmap dirs, though not the mod dirs. This is done so that you can fix any discrepancies just by using the mod syncmaps.

4. In the absence of any warnings about irreconcilable differences, when the files in A\_mod\_dir are copied back to A, and the same for B, A and B should become exact mirrors. To prove this, if necessary, make any necessary changes to A\_mod\_syncmap and B\_mod\_syncmap to reconcile any differences that raised warnings. Then run `python compare\_syncmaps.py A\_mod\_syncmap B\_mod\_syncmap --fix`. This runs *fix mode*, which re-uses the mod syncmaps and mod dirs you created previously. If the syncmaps are identical, you will get a message saying so. If, however, there are still differences, they will be reported. During fix mode, no files are actually manually changed at all (not even .hash files), except for the **root.map** file (to change the values of RESULT_OR_EMPTY entries; see the **root.map** section for details). However, if, in the original mod syncmaps, there were unreconcilable differences (which raised warnings, and were indicated in **root.map** as DIFF\_UNCHANGED), when you run fix mode, if those differences have been fixed, the files you manually altered in the syncmaps will be automatically copied into the mod dirs. Using fix mode prevents you from having to create an additional layer of mod syncmaps each time you make manual changes to fix unreconcilable differences and then verify to see if the resulting syncmaps are the same.

(Note that it is not necessary for mod syncmaps to contain any .txt or .hash files which were not marked as different from the versions present in the other input syncmap (in other words, as DIFF\_CHANGED or DIFF\_UNCHANGED, in **root.map**. However, these files will be present in the mod syncmaps, anyway.)

## Comparison guidelines

There are some guidelines that apply to how syncmaps are compared:

* Non-.txt, non-.hash files not present in the other input syncmap will be added to the appropriate mod syncmap (for example, if a file is present in A but not B, it will be added to B\_mod\_syncmap). Between existing A and B files, differences in file size (and, if available, hash value) will be noted as warnings that must be handled manually.
-Any non-.hash files whose file sizes disagree are noted as warnings, and are included in the mod syncmaps but not the mod dirs. This includes .txt files. A separate **root_warnings.txt** file is created to indicate these situations.

* For any .hash files that disagree, attempts are made to reconcile any missing files from each and produce an identical merged .hash file. If the creation of a merged hash file fails, a warning is noted and no merged hash file will be created in the mod syncmaps. If the result does not differ, copies of the same merged .hash file are made in both mod syncmaps.

* Hash value mismatches are only available when dealing with identically-named hash entries in same-named .hash files located in matching locations within the A and B dir hierarchies.

* The only files that will appear in a mod syncmap for a directory if such a file exists in the associated root directory are .hash and .txt files.

* If you run **compare\_syncmaps.py** in normal mode, it will raise an error if the mod dirs associated with the input syncmaps already exist. If you run the program in fix mode, it will raise an error if the associated mod dirs don**t exist.

## root.map

**root.map** is a text file which contains lines of the format:
`RESULT_OR_EMPTY,DIR_OR_SIZE,RELATIVE_PATH`

Example:

`,DIR,Dir,512,Dir/File.jpg`

If DIR\_OR\_SIZE is the string "DIR" instead of a size, this indicates that the entry is a directory, not a file. All entries will appear in the same order in which they are encountered when walking the root directory normally.

Any entry for a .txt or .hash file is expected to have a matching existing .txt or .hash file alongside the **root.map** file in the syncmap dir, and in the appropriate subdirectory matching the RELATIVE\_PATH. If no such file exists for a .txt or .hash file entry, a warning is shown.

Note that in the above example, the RESULT\_OR\_EMPTY is empty, because this is an input syncmap. In a mod syncmap, RESULT\_OR\_EMPTY would never be empty, and would always be either NEW, SAME, DIFF\_CHANGED, or DIFF\_UNCHANGED. If NEW, this indicates that the file is new and was not in the input syncmap; it was copied over from the other input syncmap. If DIFF\_CHANGED or DIFF\_UNCHANGED, this file is either changed or unchanged from the version present in the input syncmap (if it's changed, it must be a .hash file), but regardless, was different from the version of the file from the other input syncmap. If SAME, this file is unchanged from the version of the file from the input syncmap, and is also identical to the matching file in the other input syncmap. The only files which should cause problems requiring fixing the syncmaps, and the only files which will raise warnings, are those are those which have the type DIFF\_UNCHANGED.

Any entry for a .txt or .hash file is expected to have a matching existing .txt or .hash file alongside the **root.map** file in the syncmap dir, and in the appropriate subdirectory matching the RELATIVE\_PATH. If no such file exists for a .txt or .hash file entry, a warning is shown while running **compare\_syncmaps.py**.