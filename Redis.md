### 滑动窗口限频
```java
/**
     * 执行限频检查 - 使用滑动窗口限频
     *
     * @param req 请求参数
     */
    private void checkSlidingWinRateLimit(DiscountExportSkuReq req) {
        String acctId = String.valueOf(req.getAcctDTO().getAcctId());

        // 使用滑动窗口限频服务进行检查
        if (!slidingWindowRateLimitService.checkUserExportRateLimit(acctId)) {
            long resetTime = slidingWindowRateLimitService.getUserRateLimitResetTime(acctId);

            String message = resetTime > 0 ?
                String.format("您的操作过于频繁，请%d秒后再试", resetTime) :
                "您的操作过于频繁，请稍后再试";

            CommonLogUtil.warn(log, "用户导出触发滑动窗口限频 acctId={}, resetTime={}秒", acctId, resetTime);
            throw new ZSPTBizException(BasicErrorCodeEnum.UNKNOWN_ERROR, message);
        }

        // 成功通过限频检查后，记录剩余次数用于日志
        int remainingCount = slidingWindowRateLimitService.getUserRemainingCount(acctId);
        CommonLogUtil.info(log, "用户导出滑动窗口限频检查通过 acctId={}, remainingCount={}", acctId, remainingCount);
    }


package com.sankuai.marketinguoc.zspt.business.app.service.discount;

import com.dianping.squirrel.client.StoreKey;
import com.dianping.squirrel.client.impl.redis.RedisStoreClient;
import com.sankuai.marketinguoc.zspt.manage.infra.repository.common.MccService;
import com.sankuai.meituan.waimai.c.marketing.beans.utils.log.CommonLogUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.Set;

/**
 * 基于RedisStoreClient的滑动窗口限频服务
 */
@Slf4j
@Service
public class SlidingWindowRateLimitService {

    @Resource(name = "squirrelDBClient")
    private RedisStoreClient redisStoreClient;

    @Resource
    private MccService mccService;

    private static final String RATE_LIMIT_CATEGORY = "sliding_window_rate_limit";

    /**
     * 滑动窗口限频检查
     *
     * @param userId 用户ID
     * @return true-通过限频检查，false-触发限频
     */
    public boolean checkUserExportRateLimit(String userId) {
        if (!mccService.getDiscountExportRateLimitEnabled()) {
            return true;
        }

        StoreKey storeKey = new StoreKey(RATE_LIMIT_CATEGORY, userId);
        long now = System.currentTimeMillis();
        int timeWindow = mccService.getDiscountExportUserTimeWindow() * 1000; // 转换为毫秒
        int maxCount = mccService.getDiscountExportUserMaxCount();

        try {
            // 1. 删除时间窗口外的记录
            long windowStart = now - timeWindow;
            redisStoreClient.zremrangeByScore(storeKey, 0, windowStart);

            // 2. 统计当前时间窗口内的访问次数
            Long currentCount = redisStoreClient.zcount(storeKey, windowStart, now);

            if (currentCount >= maxCount) {
                CommonLogUtil.warn(log, "滑动窗口限频触发 userId={}, currentCount={}, maxCount={}, timeWindow={}秒",
                    userId, currentCount, maxCount, mccService.getDiscountExportUserTimeWindow());
                return false;
            }

            // 3. 记录本次访问
            redisStoreClient.zadd(storeKey, now, String.valueOf(now));

            // 4. 设置key的过期时间（防止内存泄漏）
            int expireSeconds = (timeWindow + 1000) / 1000;
            redisStoreClient.expire(storeKey, expireSeconds);

            CommonLogUtil.info(log, "滑动窗口限频检查通过 userId={}, currentCount={}, maxCount={}, timeWindow={}秒",
                userId, currentCount + 1, maxCount, mccService.getDiscountExportUserTimeWindow());
            return true;

        } catch (Exception e) {
            CommonLogUtil.error(log, "滑动窗口限频检查异常 userId={}", userId, e);
            // 异常情况下放行，避免影响正常业务
            return true;
        }
    }

    /**
     * 获取下次可以访问的时间（秒）
     * 基于最早的访问记录计算
     *
     * @param userId 用户ID
     * @return 下次可访问的等待时间（秒），0表示可以立即访问
     */
    public long getUserRateLimitResetTime(String userId) {
        if (!mccService.getDiscountExportRateLimitEnabled()) {
            return 0;
        }

        StoreKey storeKey = new StoreKey(RATE_LIMIT_CATEGORY, userId);
        long now = System.currentTimeMillis();
        int timeWindow = mccService.getDiscountExportUserTimeWindow() * 1000;

        try {
            // 获取最早的一次访问时间
            Set<String> earliest = redisStoreClient.zrange(storeKey, 0, 0);
            if (earliest == null || earliest.isEmpty()) {
                return 0; // 没有访问记录，可以立即访问
            }

            long earliestTime = Long.parseLong(earliest.iterator().next());
            long nextAvailableTime = earliestTime + timeWindow;

            return Math.max(0, (nextAvailableTime - now) / 1000);
        } catch (Exception e) {
            CommonLogUtil.error(log, "获取下次可访问时间异常 userId={}", userId, e);
            return 0;
        }
    }

    /**
     * 获取用户在滑动窗口内的剩余次数
     *
     * @param userId 用户ID
     * @return 剩余次数
     */
    public int getUserRemainingCount(String userId) {
        if (!mccService.getDiscountExportRateLimitEnabled()) {
            return Integer.MAX_VALUE;
        }

        StoreKey storeKey = new StoreKey(RATE_LIMIT_CATEGORY, userId);
        long now = System.currentTimeMillis();
        int timeWindow = mccService.getDiscountExportUserTimeWindow() * 1000;
        int maxCount = mccService.getDiscountExportUserMaxCount();

        try {
            long windowStart = now - timeWindow;
            Long currentCount = redisStoreClient.zcount(storeKey, windowStart, now);
            return Math.max(0, maxCount - currentCount.intValue());
        } catch (Exception e) {
            CommonLogUtil.error(log, "获取滑动窗口剩余次数异常 userId={}", userId, e);
            return maxCount;
        }
    }

    /**
     * 清除用户的限频记录（管理员功能）
     *
     * @param userId 用户ID
     * @return 是否清除成功
     */
    public boolean clearUserRateLimit(String userId) {
        StoreKey storeKey = new StoreKey(RATE_LIMIT_CATEGORY, userId);
        try {
            Boolean result = redisStoreClient.delete(storeKey);
            CommonLogUtil.info(log, "清除用户限频记录 userId={}, result={}", userId, result);
            return Boolean.TRUE.equals(result);
        } catch (Exception e) {
            CommonLogUtil.error(log, "清除用户限频记录异常 userId={}", userId, e);
            return false;
        }
    }
}
```
