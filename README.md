# remote-sync
Synchronize remotely stored directories.

## Description

The tools in this package allow you to sync the contents of 2 possibly remotely stored root directories, A and B, so that they will become identical. For the sake of preventing back-and-forth and repeatedly scanning the original root directories, this syncing is done indirectly through *syncmaps* and *exportdirs*.

When syncing two directories, A and B, first, you create a *syncmap* of each: a map which contains relevant metadata information for each file in a root directory (as well as direct copies of certain files). Then, you compare the two syncmaps. This updates the syncmaps with information on which files each syncmap needs to be copied from its root directory to the other syncmap's root directory, in order to make the root directories identical.

If the syncmaps have any files in common whose differences can't be automatically reconciled, you must reconcile these conflicts manually. Then you can re-run the syncmap comparison to verify that the results of applying the syncmaps will result in identical root directories. You can run a script to get the files each syncmap needs to export from its associated root directory, thus generating an **exportdir** to be applied to the other root directory.

## Usage example

A and B are two directories whose contents mostly mirror each other but may each have differences. To turn them into exact mirrors of each other, do the following steps:

1. Run: `python create_syncmap.py A` on computer which has A
2. Run: `python create_syncmap.py B` on computer which has B
3. Run: `python compare_syncmaps.py A_syncmap B_syncmap`

 If there are any files with unreconcilable differences, warnings will be shown. This occurs if there are files at the same relative paths in A and B which contain different contents. This is determined by file size, checksum if present in an existing checksum file (same-directory .hash files are currently supported), and direct file content comparisons in the case of checksum files and special *direct compare* files (by default, all .txt files).

4. If you get warnings about A and B having different files with identical names:

 i. Make manual changes to unreconcilable differences between A\_syncmap and B\_syncmap.

 ii. Run: `python compare_syncmaps.py A_syncmap B_syncmap`

 iii. Repeat

5. Run: `python create_export_dir.py A A_syncmap --del_syncmap` on computer which has A, creating A\_exportdir
6. Copy contents of A\_exportdir to B
7. Run: `python create_export_dir.py B B_syncmap --del_syncmap` on computer which has B, creating B\_exportdir
8. Copy contents of B\_exportdir to A

## Syncmaps

A *syncmap* is a map of a root directory, storing some data for each file. This map is a directory itself, and must contain a **root.map** file, containing data about each of the files in the original directory, and also data about how the synchronization process for each of those files is going.

There are certain types of *direct compare* files for which a syncmap stores not just the relative path and file size, but also the entire file. Any such file will be be referenced by the **root.map** file. Checksum files and, by default, .txt files are direct compare files.

Note that, after syncmap comparison, it is not necessary for the updated syncmaps to contain copies of any direct compare files which caused no conflicts during the comparison. However, these files will be present in the updated syncmaps, anyway.

## Exportdirs

An *exportdir* is a directory created by applying a syncmap to its associated root directory. The exportdir contains files that should be applied to the other root directory, to make the root directories match. Exportdirs also contain files which had the same names but different contents in the root directories, and which have had conflicts automatically or manually reconciled.

## root.map

**root.map** is a text file which contains lines of the format:

`STATUS,DIR_OR_SIZE,RELATIVE_PATH`

Example:

`UNCHECKED,DIR,Dir,512,Dir/File.jpg`

If DIR\_OR\_SIZE is the string "DIR" instead of a size, this indicates that the entry is a directory, not a file. All entries will appear in the same order in which they are encountered when walking the root directory normally.

Any entry for a direct compare or checksum file is expected to have a matching existing such file alongside the **root.map** file in the syncmap dir, in the appropriate location matching RELATIVE\_PATH. If no such file exists, a warning is shown.

Note that in the above example, the STATUS is UNCHECKED, because it has not been compared with another syncmap yet. After comparison, STATUS would be one of these values: EXPORT, SAME, DIFF\_UNCHANGED, or DIFF\_CHANGED.

* If EXPORT, this indicates that the file is new to the other syncmap and should be exported.
* If DIFF\_CHANGED or DIFF\_UNCHANGED, this file has either been unchanged or been automatically changed from the version present in the input syncmap (if DIFF\_CHANGED, it must be a checksum file). Regardless, this file, during comparison, was found to be different from the version of the file from the other syncmap.
* If SAME, this file is identical to the matching file in the other syncmap.

The only status type which will cause a warning, and which will also require manually fixing the syncmaps, is DIFF\_UNCHANGED.

You can only call **create\_export\_dir.py** successfully when there are no DIFF\_UNCHANGED entries in the syncmap's **root.map** file. You can optionally pass the argument **--del\_syncmap**, to delete the syncmap after creating the exportdir.

## Syncmap comparison notes

* Non-direct compare, non-checksum files not present in the other syncmap will be marked to be exported. Between existing files, differences in file size (and, if available, checksum) will be noted as warnings that must be handled manually.

* For checksums, only same-directory .hash files are currently supported. Hash value mismatches are only available when dealing with identically-named hash entries in same-named .hash files located in matching locations within the two root directory hierarchies.

* Any non-checksum files whose file sizes disagree are noted as warnings, and the two versions in the syncmaps need to be manually edited before comparison can succeed.

* For any checksum files that disagree, attempts are made to reconcile any missing files from each checksum file and produce an identical merged checksum file in each syncmap. If the creation of a merged checksum file fails, a warning is noted.

* A **root_warnings.txt** file is created to indicate things that must be reconciled before an exportdir can be created.