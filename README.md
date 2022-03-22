# RvtLock3r

Revit .NET C# add-in to validate that certain BIM element properties have not been modified.

- [Checksum](https://en.wikipedia.org/wiki/Checksum) + encryption
- [.NET MD5 Class](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.md5?view=net-6.0)
- [forge.wpf-csharp](https://github.com/Autodesk-Forge/forge.wpf.csharp/tree/secure-dev)
- For more security and harder hacking, you can grab the values, calculate the checksum, aka signature, rotate a couple of bytes, breaking the signature, and implement the algorithm to know how to restore the proper order before decrypting, 
- Original sample model: <i>Z:/Users/jta/a/special/gypsum/test/british-gypsum-bim-a206a167-en.rvt</i>
- Revit 2022 sample: <i>/Users/jta/a/special/gypsum/test/british-gypsum-bim-a206a167-en_2022.rvt</i>

## Motivation

Revit does not provide any functionality to ensure that shared parameter values are not modified.

The add-in stores a checksum for the original intended values of selected shared parameters and implements a validation function to ensure that the current values compute the same checksum.

The validation function is initially implemented as an external command.

It may later be triggered automatically on opening or saving a document to notify the user that undesired tampering has taken place.

### Model Checker Caveat

In any serious BIM environment, many rules and conventions are applied anbd required.
Tools such as the [Autodesk Model Checker](https://interoperability.autodesk.com/modelchecker.php) ensure that these are strictly followed and can be relied upon.
Maybe you should be using such a tool providing more coverage than RvtLock3r does?
]
## Validation

The customer add-in reads a set of [ground truth](https://en.wikipedia.org/wiki/Ground_truth) data from some [storage location](#storage). It contains a list of triples:

- `ElementId`
- Shared parameter `GUID`
- Checksum

The add-in iterates over all elements and shared parameters specified by these triples, reads the corresponding shared parameter value, calculates its checksum and validates it by comparison with the ground truth value.

Discrepancies are logged and a report is presented to the user.

The add-in does not care what kind of elements or shared parameters are being examined.
That worry is left up to whoever creates the ground truth file.

In the initial proof of concept, the triples are simply space separated in individual lines in a text file.

## Preparation

There are various possible approaches to prepare
the [ground truth](https://en.wikipedia.org/wiki/Ground_truth) input text file,
and they can be completely automated, more or less programmatically assisted, or fully manual.

In all three cases, you will first need to determine up front what elements and which shared parameters on them are to be checked. Retrieve the corresponding parameter values, compute their checksums, and save the above-mentioned triples.

## Storage

The ground truth data triples containing the data required for integrity validation needs to be stored somewhere. That could be hard-wired directly into the add-in code for a specific BIM, stored in an external text file, within the `RVT` document, or elsewhere; it may be `JSON` formatted; it may be encrypted; still to be decided.

Two options are available for storing custom data directly within the `RVT` project file: shared parameters and extensible storage.
The latter is more modern and explicitly tailored for use by applications and data that is not accessible to the end user or even Revit itself.
That seems most suitable for our purpose here.
Extensible storage can be added to any database element.
However, it interferes least with Revit operation when placed on a dedicated `DataStorage` element,
especially [in a worksharing environment](http://thebuildingcoder.typepad.com/blog/2015/02/extensible-storage-in-a-worksharing-environment.html).
Creation and population of a `DataStorage` element is demonstrated by the [named GUID storage for project identification](https://thebuildingcoder.typepad.com/blog/2016/04/named-guid-storage-for-project-identification.html) sample.

## Plan

Proposal:

- Get all the shared parameters of the element with their values
- Export the shared parameters and their values to a file `.txt`
- Use the SHA256 or MD5 algorithm to compute the hash of each property value and store them in the file
- Subscribe to `DocumentClosing` event
- Compare the checksums

We can use the [RevitPythonShell](https://github.com/architecture-building-systems/revitpythonshell) or `RPS` to analyse the `RVT` and export the element and parameter data of interest.
Maybe restrict to one single family, or only some types, or only some params, but that can come later.
Compute the checksum for each parameter value separately to enable reporting which element and which property has been modified, if any.
Optionally encrypt the entire file.
Use RPS only to export the list of parameters and values.

For each value, store:

- ElementId
- Parameter `Definition` ElementId
- Parameter value [AsValueString](https://www.revitapidocs.com/2022/5015755d-ee80-9d74-68d9-55effc60ed0c.htm) (not really needed)

I would implement the checksum computation and later the optional encryption in C#, not Python.

The C# add-in skeleton already implements an external app + external command.

We can implement the command to read the text file listing element and parameter ids; for each pair, open the element and its parameter, determine the value (should match the text file) and calculate the checksum. Replace the parameter value in the txt file by its checksum.
That becomes the [ground truth](https://en.wikipedia.org/wiki/Ground_truth) file that is referenced later, containing just a list of three numbers for each property to check:

- ElementId
- Parameter `Definition` ElementId
- Parameter value checksum
 
The final real-life command will read and decrypt the external ground truth file, run the code to calculate the current param value checksums, compare with the ground truth, log all discrepancies and report the result.
 
Once it works, we can trigger the validation command automatically via an event instead of manually via an external command, e.g., using the [DocumentOpening](https://www.revitapidocs.com/2022/99a0bcc4-fede-b66b-198d-a53f46ecf149.htm) and
[DocumentClosing](https://www.revitapidocs.com/2022/2f0a7a6f-ed8b-0518-c5f8-edb14b321296.htm) events.

## Authors

Carol Gitonga and 
Jeremy Tammik,
[The Building Coder](http://thebuildingcoder.typepad.com),
[Forge](http://forge.autodesk.com) [Platform](https://developer.autodesk.com) Development,
[ADN](http://www.autodesk.com/adn)
[Open](http://www.autodesk.com/adnopen),
[Autodesk Inc.](http://www.autodesk.com)

## License

This sample is licensed under the terms of the [MIT License](http://opensource.org/licenses/MIT).
Please see the [LICENSE](LICENSE) file for full details.
