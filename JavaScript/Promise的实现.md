```javaScript
class Promise {
  constructor(immediateFunc) {
    this.status = 'pending';
    this.value = undefined;
    this.err = undefined;
    this.resolveCallbackList = [];
    this.rejectCallbackList = [];
    const self = this;

    const resolve = function (value) {
      if (self.status === 'pending') {
        self.value = value;
        self.status = 'fulfilled';
        self.resolveCallbackList.forEach((resolveCallback) => {
          resolveCallback(value);
        });
        self.rejectCallbackList = [];
      }
    };

    const reject = function (err) {
      if (self.status === 'pending') {
        self.err = err;
        self.status = 'rejected';
        let callbackLen = self.rejectCallbackList.length;
        self.rejectCallbackList.forEach((rejectCallback) => {
          rejectCallback(err);
        });
        self.rejectCallbackList = [];
        // await nextTick;
        if (!callbackLen) {
          throw err;
        }
      }
    }

    try {
      immediateFunc(resolve, reject);
    } catch (e) {
      reject(e);
    }
  }

  resolvePromise(curPromise, oriPromise, resolve, reject) {
    if (curPromise === oriPromise) {
      return reject(new TypeError('循环引用'));
    }
    if (oriPromise instanceof Promise || (oriPromise !== null && (typeof oriPromise === 'object' || typeof oriPromise === 'function') && oriPromise.then)) {
      oriPromise.then((value) => {
        this.resolvePromise(curPromise, value, resolve, reject);
      }, reject);
    } else {
      return resolve(oriPromise);
    }
  }

  then(resolveCallback, rejectCallback) {
    const self = this;
    const resPromise = new Promise((resolve, reject) => {
      const resolveCallWrap = function (value) {
        // await nextTick
        try {
          const resolvePromise = resolveCallback ? resolveCallback(value) : () => value;
          self.resolvePromise(resPromise, resolvePromise, resolve, reject);
        } catch (e) {
          reject(e);
        }
      };
      const rejectCallbackWrap = function (err) {
        // await nextTick
        try {
          const rejectPromise = rejectCallback ? rejectCallback(err) : (err) => { throw err };
          self.resolvePromise(resPromise, rejectPromise, resolve, reject);
        } catch (e) {
          reject(e);
        }
      };
      if (self.status === 'pending') {
        self.resolveCallbackList.push(resolveCallWrap);
        self.rejectCallbackList.push(rejectCallbackWrap);
      }
      if (self.status === 'fulfilled') {
        resolveCallWrap(self.value);
      }
      if (self.status === 'rejected') {
        rejectCallbackWrap(self.err);
      }
    });

    return resPromise;
  }

  catch(rejectCallback) {
    return this.then(undefined, rejectCallback);
  }

  static resolve(oriPromise) {
    if (oriPromise instanceof Promise) {
      return oriPromise;
    } else if (oriPromise !== null && (typeof oriPromise === 'object' || typeof oriPromise === 'function') && oriPromise.then) {
      let resPromise = new Promise((resolve, reject) => {
        this.resolvePromise(resPromise, oriPromise, resolve, reject);
      });
      return resPromise;
    } else {
      return new Promise((resolve) => resolve(oriPromise));
    }
  }

  static reject(reason) {
    return new Promise((resolve, reject) => reject(reason));
  }

  finally(callback) {
    const resolveCallback = function (value) {
      Promise.resolve(callback()).then(() => value);
    };
    const rejectCallback = function () {
      Promise.resolve(callback()).then(() => {
        throw err;
      });
    }
    return this.then(resolveCallback, rejectCallback)
  }
}

let a = new Promise((resolve, reject) => {
  setTimeout(() => {
    // resolve('adasdsd');
    reject('asdasd');
  }, 300);
})

a.then((result) => {
  console.log(result);
}, (err) => {
  console.log(err);
})

console.log(1)


```