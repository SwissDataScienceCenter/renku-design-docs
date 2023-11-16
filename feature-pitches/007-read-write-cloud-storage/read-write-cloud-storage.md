# Make Cloud Storage Read/Write (+ “Universal” Storage)

Authors: Laura Kinkead, Ralf Grubenmann, Tasko Olevski, Mohammad Alisafaee

---

## 🤔 Problem

Primary issue: Users are confused and frustrated when they hear they can only read from S3 buckets, and can’t write to them.

Secondary issue: Users want to be able to connect to data in other storage services other than S3 (for example, PolyBox). This is particularly important to the SDSC Academic Team (according to Nati).

Also datashim (the current technology for mounting S3) is unrealiable and has lots of issues (r/w not working properly, sometimes doesn’t mount, not really actively supported)

## 🚞 User stories / journeys

As a RenkuLab sessions user, I want to be able to write data outputs back to my S3 bucket.

As a SDSC Academic Team member, I want to mount my NextCloud or PolyBox drive in a RenkuLab session so I have some personal storage space that I take with me across all my projects.

## 🍴 Appetite

6 weeks.

## 🎯 Solution

### User Flows

User should be able to select read+write mode when configuring cloud storage (and this should be the default).

User should be able to read and write to the mounted bucket in their session like normal.

When configuring cloud storage, the user should be able to select which storage type they are adding, and be brought to the relevant form for that type. For some types (S3), we provide a basic entry form, and for all others we ask the user to enter an rclone config. The build team should work with Product to determine if we want (i.e. have time) to provide an “easy” mode for any additional storage types. The top priority additional storage types are:

- PolyBox/NextCloud/OwnCloud (WebDAV)
- Google Drive (may require that we make a Renku app)
- Dropbox (may require that we make a Renku app)

We will probably need good quality documentation to help users use many of these.

### Technical Details

As we already store RClone configurations in the storage service, we want a CSI driver that can directly mount RClone storage, so supporting all the same storage that RClone supports. RClone itself is mature technology and a driver built on top of it should benefit from that maturity as well. E.g. [https://github.com/wunderio/csi-rclone](https://github.com/wunderio/csi-rclone)

We need to test the RClone CSI driver to verify that it works as expected.

We need to set up the RClone CSI driver, as part of the Renku Helm chart.

We need to change the notebooks service to send persistent volume patches to Amalthea that conform to the new driver instead of the datashim CRDs.

## 🐰 Rabbit Holes

We haven’t used this CSI driver before.

The CSI driver is maintained, but maintainers aren’t responsive.

The CSI Driver does not have very detailed documentation.

The CSI driver needs a globally configured secret, which likely can be empty, but we might need to make our own patched version of it without this.

## 🙅‍♀️ No-gos

This build is separate from Secret Storage, meaning it does not include saving any storage credentials.
