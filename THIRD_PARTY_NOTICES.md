# Third-Party Notices

The ION Android SDK ("ION SDK") by Hidonix Industries includes, or depends on, the third-party software listed below. Each component is governed by its own license, reproduced or referenced in this file. These notices do not change the terms under which the ION SDK itself is licensed.

- [1. Components bundled in the SDK artifact](#1-components-bundled-in-the-sdk-artifact)
- [2. Runtime dependencies](#2-runtime-dependencies)
- [3. Notes for integrators](#3-notes-for-integrators)
- [4. License texts](#4-license-texts)

---

## 1. Components bundled in the SDK artifact

These components are compiled or packaged into the ION SDK AAR itself.

| Component | Version | Location in the AAR | License |
| --- | --- | --- | --- |
| [VQF](https://github.com/dlaidig/vqf) — Versatile Quaternion-based Filter, © 2021 Daniel Laidig | 2.1.2 | `jni/*/libvqf-shadow.so` | [MIT](#mit-license--vqf) |
| [LLVM libc++ / libc++abi](https://libcxx.llvm.org/) (Android NDK C++ runtime, linked statically) | Android NDK r28 | `jni/*/libstocor-android.so`, `jni/*/libvqf-shadow.so` | [Apache-2.0 WITH LLVM-exception](#apache-license-20) |
| Android NDK `native_app_glue`, © 2010 The Android Open Source Project | Android NDK r28 | `jni/*/libstocor-android.so` | [Apache-2.0](#apache-license-20) |
| [ARCore SDK for Android](https://github.com/google-ar/arcore-android-sdk) sample shaders, © 2017–2022 Google LLC | — | `assets/shaders/*.vert`, `assets/shaders/*.frag` | [Apache-2.0](#apache-license-20) |

The copyright and license headers of the VQF sources and of the ARCore shaders are preserved in the files themselves.

---

## 2. Runtime dependencies

These libraries are **not** bundled in the AAR. They are resolved by the integrating application (see the dependency list in the README), and are listed here because the ION SDK requires them at runtime. The version is the one the SDK is built against; the application may resolve a newer compatible one.

| Library | Maven coordinates | Version | License |
| --- | --- | --- | --- |
| Kotlin Standard Library | `org.jetbrains.kotlin:kotlin-stdlib` | 2.0.21 | [Apache-2.0](#apache-license-20) |
| kotlinx.coroutines | `org.jetbrains.kotlinx:kotlinx-coroutines-android` | 1.7.3 | [Apache-2.0](#apache-license-20) |
| kotlinx.serialization | `org.jetbrains.kotlinx:kotlinx-serialization-json` | 1.6.0 | [Apache-2.0](#apache-license-20) |
| Retrofit | `com.squareup.retrofit2:retrofit` | 2.9.0 | [Apache-2.0](#apache-license-20) |
| Retrofit Gson converter | `com.squareup.retrofit2:converter-gson` | 2.7.2 | [Apache-2.0](#apache-license-20) |
| OkHttp | `com.squareup.okhttp3:okhttp` | 4.12.0 | [Apache-2.0](#apache-license-20) ¹ |
| OkHttp Logging Interceptor | `com.squareup.okhttp3:logging-interceptor` | 4.11.0 | [Apache-2.0](#apache-license-20) |
| Okio (transitive, via OkHttp) | `com.squareup.okio:okio` | — | [Apache-2.0](#apache-license-20) |
| Gson | `com.google.code.gson:gson` | 2.8.9 | [Apache-2.0](#apache-license-20) |
| FlatBuffers | `com.google.flatbuffers:flatbuffers-java` | 25.2.10 | [Apache-2.0](#apache-license-20) |
| TensorFlow Lite | `org.tensorflow:tensorflow-lite` | 2.17.0 | [Apache-2.0](#apache-license-20) |
| Android Beacon Library | `org.altbeacon:android-beacon-library` | 2.19.5 | [Apache-2.0](#apache-license-20) |
| AndroidX AppCompat (and transitive AndroidX Core, Annotation, Lifecycle) | `androidx.appcompat:appcompat` | 1.6.1 | [Apache-2.0](#apache-license-20) |
| Google Play services – Location | `com.google.android.gms:play-services-location` | 17.1.0 | [Android Software Development Kit License](https://developer.android.com/studio/terms) |
| ARCore (Google Play Services for AR) | `com.google.ar:core` | 1.51.0 | [ARCore Additional Terms of Service](https://developers.google.com/ar/develop/terms) |

¹ OkHttp embeds a copy of the [Public Suffix List](https://publicsuffix.org/), which is licensed under the [Mozilla Public License 2.0](https://mozilla.org/MPL/2.0/).

---

## 3. Notes for integrators

- **ARCore.** ARCore is distributed by Google under the ARCore Additional Terms of Service, not under an open-source license. If your application enables AR-assisted navigation, those terms apply to your application too, including their requirement to disclose to users that the application uses Google Play Services for AR (ARCore). Review the [current terms](https://developers.google.com/ar/develop/terms) before publishing.
- **Google Play services.** `play-services-location` is proprietary software distributed by Google under the Android Software Development Kit License.
- **Attribution.** If your application shows open-source notices to its users (for example with the [OSS Licenses Gradle Plugin](https://developers.google.com/android/guides/opensource)), the runtime dependencies above are picked up automatically from your dependency graph. The components in [section 1](#1-components-bundled-in-the-sdk-artifact) are inside the AAR and are not: include this file, or its relevant entries, in your notices.

---

## 4. License texts

### MIT License — VQF

```
MIT License

Copyright (c) 2021 Daniel Laidig <laidig@control.tu-berlin.de>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

### Apache License 2.0

Applies to every component marked Apache-2.0 above.

```
                              Apache License
                        Version 2.0, January 2004
                     http://www.apache.org/licenses/

TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

1. Definitions.

   "License" shall mean the terms and conditions for use, reproduction,
   and distribution as defined by Sections 1 through 9 of this document.

   "Licensor" shall mean the copyright owner or entity authorized by
   the copyright owner that is granting the License.

   "Legal Entity" shall mean the union of the acting entity and all
   other entities that control, are controlled by, or are under common
   control with that entity. For the purposes of this definition,
   "control" means (i) the power, direct or indirect, to cause the
   direction or management of such entity, whether by contract or
   otherwise, or (ii) ownership of fifty percent (50%) or more of the
   outstanding shares, or (iii) beneficial ownership of such entity.

   "You" (or "Your") shall mean an individual or Legal Entity
   exercising permissions granted by this License.

   "Source" form shall mean the preferred form for making modifications,
   including but not limited to software source code, documentation
   source, and configuration files.

   "Object" form shall mean any form resulting from mechanical
   transformation or translation of a Source form, including but
   not limited to compiled object code, generated documentation,
   and conversions to other media types.

   "Work" shall mean the work of authorship, whether in Source or
   Object form, made available under the License, as indicated by a
   copyright notice that is included in or attached to the work
   (an example is provided in the Appendix below).

   "Derivative Works" shall mean any work, whether in Source or Object
   form, that is based on (or derived from) the Work and for which the
   editorial revisions, annotations, elaborations, or other modifications
   represent, as a whole, an original work of authorship. For the purposes
   of this License, Derivative Works shall not include works that remain
   separable from, or merely link (or bind by name) to the interfaces of,
   the Work and Derivative Works thereof.

   "Contribution" shall mean any work of authorship, including
   the original version of the Work and any modifications or additions
   to that Work or Derivative Works thereof, that is intentionally
   submitted to Licensor for inclusion in the Work by the copyright owner
   or by an individual or Legal Entity authorized to submit on behalf of
   the copyright owner. For the purposes of this definition, "submitted"
   means any form of electronic, verbal, or written communication sent
   to the Licensor or its representatives, including but not limited to
   communication on electronic mailing lists, source code control systems,
   and issue tracking systems that are managed by, or on behalf of, the
   Licensor for the purpose of discussing and improving the Work, but
   excluding communication that is conspicuously marked or otherwise
   designated in writing by the copyright owner as "Not a Contribution."

   "Contributor" shall mean Licensor and any individual or Legal Entity
   on behalf of whom a Contribution has been received by Licensor and
   subsequently incorporated within the Work.

2. Grant of Copyright License. Subject to the terms and conditions of
   this License, each Contributor hereby grants to You a perpetual,
   worldwide, non-exclusive, no-charge, royalty-free, irrevocable
   copyright license to reproduce, prepare Derivative Works of,
   publicly display, publicly perform, sublicense, and distribute the
   Work and such Derivative Works in Source or Object form.

3. Grant of Patent License. Subject to the terms and conditions of
   this License, each Contributor hereby grants to You a perpetual,
   worldwide, non-exclusive, no-charge, royalty-free, irrevocable
   (except as stated in this section) patent license to make, have made,
   use, offer to sell, sell, import, and otherwise transfer the Work,
   where such license applies only to those patent claims licensable
   by such Contributor that are necessarily infringed by their
   Contribution(s) alone or by combination of their Contribution(s)
   with the Work to which such Contribution(s) was submitted. If You
   institute patent litigation against any entity (including a
   cross-claim or counterclaim in a lawsuit) alleging that the Work
   or a Contribution incorporated within the Work constitutes direct
   or contributory patent infringement, then any patent licenses
   granted to You under this License for that Work shall terminate
   as of the date such litigation is filed.

4. Redistribution. You may reproduce and distribute copies of the
   Work or Derivative Works thereof in any medium, with or without
   modifications, and in Source or Object form, provided that You
   meet the following conditions:

   (a) You must give any other recipients of the Work or
       Derivative Works a copy of this License; and

   (b) You must cause any modified files to carry prominent notices
       stating that You changed the files; and

   (c) You must retain, in the Source form of any Derivative Works
       that You distribute, all copyright, patent, trademark, and
       attribution notices from the Source form of the Work,
       excluding those notices that do not pertain to any part of
       the Derivative Works; and

   (d) If the Work includes a "NOTICE" text file as part of its
       distribution, then any Derivative Works that You distribute must
       include a readable copy of the attribution notices contained
       within such NOTICE file, excluding those notices that do not
       pertain to any part of the Derivative Works, in at least one
       of the following places: within a NOTICE text file distributed
       as part of the Derivative Works; within the Source form or
       documentation, if provided along with the Derivative Works; or,
       within a display generated by the Derivative Works, if and
       wherever such third-party notices normally appear. The contents
       of the NOTICE file are for informational purposes only and
       do not modify the License. You may add Your own attribution
       notices within Derivative Works that You distribute, alongside
       or as an addendum to the NOTICE text from the Work, provided
       that such additional attribution notices cannot be construed
       as modifying the License.

   You may add Your own copyright statement to Your modifications and
   may provide additional or different license terms and conditions
   for use, reproduction, or distribution of Your modifications, or
   for any such Derivative Works as a whole, provided Your use,
   reproduction, and distribution of the Work otherwise complies with
   the conditions stated in this License.

5. Submission of Contributions. Unless You explicitly state otherwise,
   any Contribution intentionally submitted for inclusion in the Work
   by You to the Licensor shall be under the terms and conditions of
   this License, without any additional terms or conditions.
   Notwithstanding the above, nothing herein shall supersede or modify
   the terms of any separate license agreement you may have executed
   with Licensor regarding such Contributions.

6. Trademarks. This License does not grant permission to use the trade
   names, trademarks, service marks, or product names of the Licensor,
   except as required for reasonable and customary use in describing the
   origin of the Work and reproducing the content of the NOTICE file.

7. Disclaimer of Warranty. Unless required by applicable law or
   agreed to in writing, Licensor provides the Work (and each
   Contributor provides its Contributions) on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
   implied, including, without limitation, any warranties or conditions
   of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A
   PARTICULAR PURPOSE. You are solely responsible for determining the
   appropriateness of using or redistributing the Work and assume any
   risks associated with Your exercise of permissions under this License.

8. Limitation of Liability. In no event and under no legal theory,
   whether in tort (including negligence), contract, or otherwise,
   unless required by applicable law (such as deliberate and grossly
   negligent acts) or agreed to in writing, shall any Contributor be
   liable to You for damages, including any direct, indirect, special,
   incidental, or consequential damages of any character arising as a
   result of this License or out of the use or inability to use the
   Work (including but not limited to damages for loss of goodwill,
   work stoppage, computer failure or malfunction, or any and all
   other commercial damages or losses), even if such Contributor
   has been advised of the possibility of such damages.

9. Accepting Warranty or Additional Liability. While redistributing
   the Work or Derivative Works thereof, You may choose to offer,
   and charge a fee for, acceptance of support, warranty, indemnity,
   or other liability obligations and/or rights consistent with this
   License. However, in accepting such obligations, You may act only
   on Your own behalf and on Your sole responsibility, not on behalf
   of any other Contributor, and only if You agree to indemnify,
   defend, and hold each Contributor harmless for any liability
   incurred by, or claims asserted against, such Contributor by reason
   of your accepting any such warranty or additional liability.

END OF TERMS AND CONDITIONS
```

### LLVM Exceptions to the Apache 2.0 License

Applies, in addition to the Apache License 2.0, to LLVM libc++ / libc++abi.

```
As an exception, if, as a result of your compiling your source code, portions
of this Software are embedded into an Object form of such source code, you
may redistribute such embedded portions in such Object form without complying
with the conditions of Sections 4(a), 4(b) and 4(d) of the License.

In addition, if you combine or link compiled forms of this Software with
software that is licensed under the GPLv2 ("Combined Software") and if a
court of competent jurisdiction determines that the patent provision (Section
3), the indemnity provision (Section 9) or other Section of the License
conflicts with the conditions of the GPLv2, you may retroactively and
prospectively choose to deem waived or otherwise exclude such Section(s) of
the License, but only in their entirety and only with respect to the Combined
Software.
```
